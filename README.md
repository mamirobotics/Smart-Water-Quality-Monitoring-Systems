Smart Water Quality Monitoring System 💧

Sensors:
These sensor are not hardcode you uses these. According to your project use them.These sesnors have compatible for learning purpose not use real time in any induatrial setup.
For industrial setup youn use the PLC's(programmable logical controllers) with its related sensors, and actuators. I hope you understand my logic
1.pH Sensor
2.Turbidity Sensor
3.Temperature Sensor
4.Water Flow Sensor
5.TDS Sensor

Simulation provide what?
Its just teach you help you wiring and some things about the components.For further need to be buy according to your budget more components slection
so carefully. Its not final slection its may be more other best.

Programming:
sudo pip3 install adafruit-circuitpython-mcp3xxx w1thermsensor

Code:
import tkinter as tk
from tkinter import ttk
import time
import threading
import busio
import digitalio
import board
import RPi.GPIO as GPIO
from adafruit_mcp3xxx.mcp3008 import MCP3008
from adafruit_mcp3xxx.analog_in import AnalogIn
from w1thermsensor import W1ThermSensor

# Hardware Configuration
# 1. Initialize SPI and MCP3008 ADC
spi = busio.SPI(clock=board.SCK, MOSI=board.MOSI, MISO=board.MISO)
cs = digitalio.DigitalInOut(board.D8)  # CE0 / Pin 24
mcp = MCP3008(spi, cs)

# Assign MCP3008 Channels
ph_channel = AnalogIn(mcp, MCP3008.CH0)
tds_channel = AnalogIn(mcp, MCP3008.CH1)
turb_channel = AnalogIn(mcp, MCP3008.CH2)

# 2. Initialize DS18B20 Temperature Sensor
try:
    temp_sensor = W1ThermSensor()
except Exception:
    temp_sensor = None  # Fallback if sensor isn't wired/found

# 3. Initialize Water Flow Sensor (GPIO 27)
FLOW_PIN = 27
GPIO.setmode(GPIO.BCM)
GPIO.setup(FLOW_PIN, GPIO.IN, pull_up_down=GPIO.PUD_UP)

pulse_count = 0
def pulse_callback(channel):
    global pulse_count
    pulse_count += 1

GPIO.add_event_detect(FLOW_PIN, GPIO.FALLING, callback=pulse_callback)

# --- Data Processing Logic ---
def get_sensor_data():
    global pulse_count
    
    # 1. Read pH Value
    ph_voltage = ph_channel.voltage
    # Standard calibration equation: pH 7 = ~2.5V, updates based on sensor type
    ph_val = 3.5 * ph_voltage + 0.0  
    ph_val = max(0.0, min(14.0, ph_val)) # Clamp between 0 and 14

    # 2. Read TDS Value
    tds_voltage = tds_channel.voltage
    # Compensation equation for analog TDS meter
    tds_val = (133.42 * (tds_voltage**3) - 255.86 * (tds_voltage**2) + 857.39 * tds_voltage) * 0.5
    tds_val = max(0.0, tds_val)

    # 3. Read Turbidity Value
    turb_voltage = turb_channel.voltage
    # Simple conversion mapping: high voltage = clear water, low voltage = muddy water
    if turb_voltage < 2.5:
        turb_val = 3000.0
    else:
        turb_val = -1120.4 * (turb_voltage**2) + 5742.3 * turb_voltage - 4353.8
    turb_val = max(0.0, turb_val)

    # 4. Read Temperature
    if temp_sensor:
        try:
            temp_val = temp_sensor.get_temperature()
        except Exception:
            temp_val = 0.0
    else:
        temp_val = 0.0

    # 5. Read Flow Rate (Liters per minute)
    # Frequency = pulse_count over the last 1 second interval. Q = F / 7.5
    flow_rate = pulse_count / 7.5  
    pulse_count = 0 # Reset pulse counter for the next second

    # Determine Contamination Status based on WHO Standards
    is_unsafe = (ph_val < 6.5 or ph_val > 8.5 or tds_val > 5
