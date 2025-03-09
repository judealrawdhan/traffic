#!/usr/bin/env python3
import RPi.GPIO as GPIO
import time
import threading
from imx500_classification_demo import check_ambulance

# GPIO Setup
GPIO.setmode(GPIO.BCM)
GPIO.setwarnings(False)
RED_PIN = 17
YELLOW_PIN = 27
GREEN_PIN = 22
GPIO.setup([RED_PIN, YELLOW_PIN, GREEN_PIN], GPIO.OUT, initial=GPIO.LOW)

# State variables
ambulance_detected = False
state_lock = threading.Lock()

def set_lights(red, yellow, green):
    GPIO.output(RED_PIN, red)
    GPIO.output(YELLOW_PIN, yellow)
    GPIO.output(GREEN_PIN, green)

def detection_thread():
    global ambulance_detected
    while True:
        if check_ambulance():
            with state_lock:
                ambulance_detected = True
        time.sleep(0.1)

def normal_cycle():
    set_lights(GPIO.LOW, GPIO.LOW, GPIO.HIGH)  # Green
    time.sleep(15)
    set_lights(GPIO.LOW, GPIO.HIGH, GPIO.LOW)  # Yellow
    time.sleep(3)
    set_lights(GPIO.HIGH, GPIO.LOW, GPIO.LOW)  # Red
    time.sleep(15)

def main_control():
    global ambulance_detected
    detection = threading.Thread(target=detection_thread, daemon=True)
    detection.start()

    try:
        while True:
            with state_lock:
                if ambulance_detected:
                    print("Ambulance detected - GREEN LIGHT ON")
                    set_lights(GPIO.LOW, GPIO.LOW, GPIO.HIGH)
                    time.sleep(15)
                    set_lights(GPIO.LOW, GPIO.HIGH, GPIO.LOW)
                    time.sleep(3)
                    set_lights(GPIO.HIGH, GPIO.LOW, GPIO.LOW)
                    time.sleep(15)
                    ambulance_detected = False
                else:
                    normal_cycle()
    except KeyboardInterrupt:
        GPIO.cleanup()
        print("Program stopped")

if __name__ == "__main__":
    main_control()
