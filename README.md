#!/usr/bin/env python3
import RPi.GPIO as GPIO
import time
import threading
from imx500_fixed import check_ambulance

GPIO.setmode(GPIO.BCM)
GPIO.setwarnings(False)
PINS = {'red': 17, 'yellow': 27, 'green': 22}

for pin in PINS.values():
    GPIO.setup(pin, GPIO.OUT)
    GPIO.output(pin, GPIO.LOW)

state_lock = threading.Lock()
emergency_flag = False

def set_lights(red, yellow, green):
    GPIO.output(PINS['red'], red)
    GPIO.output(PINS['yellow'], yellow)
    GPIO.output(PINS['green'], green)

def detection_thread():
    global emergency_flag
    while True:
        if check_ambulance():
            with state_lock:
                emergency_flag = True
        time.sleep(0.1)

def normal_cycle():
    set_lights(0, 0, 1)  # Green
    time.sleep(15)
    set_lights(0, 1, 0)  # Yellow
    time.sleep(3)
    set_lights(1, 0, 0)  # Red
    time.sleep(15)

def emergency_cycle():
    set_lights(0, 0, 1)  # Green
    print("EMERGENCY: Ambulance detected!")
    time.sleep(20)
    set_lights(0, 1, 0)  # Yellow
    time.sleep(3)
    set_lights(1, 0, 0)  # Red
    time.sleep(10)

def main():
    detector = threading.Thread(target=detection_thread, daemon=True)
    detector.start()

    try:
        while True:
            with state_lock:
                if emergency_flag:
                    emergency_cycle()
                    emergency_flag = False
                else:
                    normal_cycle()
    except KeyboardInterrupt:
        set_lights(0, 0, 0)
        GPIO.cleanup()
        print("System stopped")

if __name__ == "__main__":
    print("Traffic Control System Active")
    main()
