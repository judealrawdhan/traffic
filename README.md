import argparse
import sys
import time
from typing import List

import cv2
import numpy as np
import threading
import RPi.GPIO as GPIO
from picamera2 import CompletedRequest, MappedArray, Picamera2
from picamera2.devices import IMX500
from picamera2.devices.imx500 import NetworkIntrinsics
from picamera2.devices.imx500.postprocess import softmax

# GPIO Configuration
PINS = {'red': 17, 'yellow': 27, 'green': 22}
RED_TIME = 30  # Seconds
GREEN_TIME = 30  # Seconds
YELLOW_TIME = 3  # Seconds

global ambulance_detected
ambulance_detected = False
system_lock = threading.Lock()

# GPIO Initialization
GPIO.setmode(GPIO.BCM)
GPIO.setwarnings(False)
for pin in PINS.values():
    GPIO.setup(pin, GPIO.OUT)
    GPIO.output(pin, GPIO.LOW)

# Camera and Model Setup
LABELS = None
last_detections = []

def get_label(request: CompletedRequest, idx: int) -> str:
    global LABELS
    if LABELS is None:
        LABELS = intrinsics.labels
        assert len(LABELS) in [1000, 1001], "Labels file should contain 1000 or 1001 labels."
        output_tensor_size = imx500.get_output_shapes(request.get_metadata())[0][0]
        if output_tensor_size == 1000:
            LABELS = LABELS[1:]
    return LABELS[idx]

class Classification:
    def __init__(self, idx: int, score: float):
        self.idx = idx
        self.score = score

def parse_classification_results(request: CompletedRequest) -> List[Classification]:
    global last_detections, ambulance_detected
    np_outputs = imx500.get_outputs(request.get_metadata())
    if np_outputs is None:
        return last_detections
    np_output = np_outputs[0]
    if intrinsics.softmax:
        np_output = softmax(np_output)
    top_indices = np.argpartition(-np_output, 3)[:3]
    top_indices = top_indices[np.argsort(-np_output[top_indices])]
    last_detections = [Classification(index, np_output[index]) for index in top_indices]
    
    # Check for ambulance detection
    for result in last_detections:
        label = get_label(request, result.idx).lower()
        if "ambulance" in label and result.score > 0.7:
            with system_lock:
                ambulance_detected = True
                print("🚨 AMBULANCE DETECTED! Switching to GREEN for emergency.")
    return last_detections

def traffic_control():
    global ambulance_detected
    try:
        while True:
            with system_lock:
                current_state = ambulance_detected
                ambulance_detected = False
            
            if current_state:
                GPIO.output(PINS['red'], GPIO.LOW)
                GPIO.output(PINS['yellow'], GPIO.LOW)
                GPIO.output(PINS['green'], GPIO.HIGH)
                print("🚦 EMERGENCY MODE: GREEN LIGHT FOR AMBULANCE")
                time.sleep(GREEN_TIME)
                GPIO.output(PINS['green'], GPIO.LOW)
                GPIO.output(PINS['yellow'], GPIO.HIGH)
                time.sleep(YELLOW_TIME)
                GPIO.output(PINS['yellow'], GPIO.LOW)
                GPIO.output(PINS['red'], GPIO.HIGH)
                time.sleep(RED_TIME)
            else:
                GPIO.output(PINS['green'], GPIO.HIGH)
                print("🚦 NORMAL TRAFFIC: GREEN LIGHT")
                time.sleep(GREEN_TIME)
                GPIO.output(PINS['green'], GPIO.LOW)
                GPIO.output(PINS['yellow'], GPIO.HIGH)
                print("🚦 NORMAL TRAFFIC: YELLOW LIGHT")
                time.sleep(YELLOW_TIME)
                GPIO.output(PINS['yellow'], GPIO.LOW)
                GPIO.output(PINS['red'], GPIO.HIGH)
                print("🚦 NORMAL TRAFFIC: RED LIGHT")
                time.sleep(RED_TIME)
                GPIO.output(PINS['red'], GPIO.LOW)
    except KeyboardInterrupt:
        GPIO.cleanup()

def get_args():
    parser = argparse.ArgumentParser()
    parser.add_argument("--model", type=str, default="/usr/share/imx500-models/imx500_network_mobilenet_v2.rpk")
    parser.add_argument("--fps", type=int, help="Frames per second")
    parser.add_argument("-s", "--softmax", action=argparse.BooleanOptionalAction)
    parser.add_argument("-r", "--preserve-aspect-ratio", action=argparse.BooleanOptionalAction)
    parser.add_argument("--labels", type=str, help="Path to the labels file")
    parser.add_argument("--print-intrinsics", action="store_true")
    return parser.parse_args()

if __name__ == "__main__":
    args = get_args()
    imx500 = IMX500(args.model)
    intrinsics = imx500.network_intrinsics
    if not intrinsics:
        intrinsics = NetworkIntrinsics()
        intrinsics.task = "classification"
    elif intrinsics.task != "classification":
        print("Network is not a classification task", file=sys.stderr)
        exit()
    
    for key, value in vars(args).items():
        if key == 'labels' and value is not None:
            with open(value, 'r') as f:
                intrinsics.labels = f.read().splitlines()
        elif hasattr(intrinsics, key) and value is not None:
            setattr(intrinsics, key, value)
    
    if intrinsics.labels is None:
        with open("assets/imagenet_labels.txt", "r") as f:
            intrinsics.labels = f.read().splitlines()
    intrinsics.update_with_defaults()
    
    if args.print_intrinsics:
        print(intrinsics)
        exit()
    
    picam2 = Picamera2(imx500.camera_num)
    config = picam2.create_preview_configuration(controls={"FrameRate": intrinsics.inference_rate}, buffer_count=12)
    
    imx500.show_network_fw_progress_bar()
    picam2.start(config, show_preview=True)
    if intrinsics.preserve_aspect_ratio:
        imx500.set_auto_aspect_ratio()
    picam2.pre_callback = parse_classification_results
    
    cam_thread = threading.Thread(target=traffic_control, daemon=True)
    cam_thread.start()
    
    while True:
        time.sleep(0.5)
