#!/usr/bin/env python3
import RPi.GPIO as GPIO
import time
import threading
from imx500_fixed import check_ambulance

# GPIO Configuration
GPIO.setmode(GPIO.BCM)
RED_PIN = 17    # GPIO17 (Physical Pin 11)
YELLOW_PIN = 27  # GPIO27 (Physical Pin 13)
GREEN_PIN = 22   # GPIO22 (Physical Pin 15)

# Initialize GPIO
GPIO.setwarnings(False)
GPIO.setup([RED_PIN, YELLOW_PIN, GREEN_PIN], GPIO.OUT, initial=GPIO.LOW)

# State Management
ambulance_detected = False
state_lock = threading.Lock()

def set_lights(red, yellow, green):
    """Control traffic light LEDs"""
    GPIO.output(RED_PIN, red)
    GPIO.output(YELLOW_PIN, yellow)
    GPIO.output(GREEN_PIN, green)

def detection_thread():
    """Monitor for ambulance detection"""
    global ambulance_detected
    while True:
        if check_ambulance():  # From camera script
            with state_lock:
                ambulance_detected = True
                print("AMBULANCE FLAG SET")
        time.sleep(0.1)  # Check 10x/second

def normal_cycle():
    """Regular traffic light sequence"""
    print("NORMAL: GREEN 15s")
    set_lights(GPIO.LOW, GPIO.LOW, GPIO.HIGH)
    time.sleep(15)
    
    print("NORMAL: YELLOW 3s")
    set_lights(GPIO.LOW, GPIO.HIGH, GPIO.LOW)
    time.sleep(3)
    
    print("NORMAL: RED 15s")
    set_lights(GPIO.HIGH, GPIO.LOW, GPIO.LOW)
    time.sleep(15)

def emergency_cycle():
    """Priority sequence for ambulance"""
    print("EMERGENCY: GREEN NOW")
    set_lights(GPIO.LOW, GPIO.LOW, GPIO.HIGH)
    time.sleep(15)  # Extended green
    
    print("EMERGENCY: YELLOW 3s")
    set_lights(GPIO.LOW, GPIO.HIGH, GPIO.LOW)
    time.sleep(3)
    
    print("EMERGENCY: RED 15s")
    set_lights(GPIO.HIGH, GPIO.LOW, GPIO.LOW)
    time.sleep(15)

def main_control():
    global ambulance_detected
    detection = threading.Thread(target=detection_thread, daemon=True)
    detection.start()

    try:
        while True:
            with state_lock:
                if ambulance_detected:
                    emergency_cycle()
                    ambulance_detected = False
                else:
                    normal_cycle()
    except KeyboardInterrupt:
        GPIO.cleanup()
        print("\nTraffic system stopped safely")

if __name__ == "__main__":
    print("Traffic Control System Started")
    main_control()
