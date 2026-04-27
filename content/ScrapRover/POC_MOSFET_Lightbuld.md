---
title: "Milestone: MOSFET PWM Control — Making a Light Bulb Pulsate from the Pi"
date: 2026-04-22
draft: false
tags: ["electronics", "raspberry-pi", "mosfet", "pwm", "motor-control", "hardware"]
summary: "First proof-of-concept for GPIO-driven power switching: using an N-channel MOSFET and PWM from the Raspberry Pi 5 to control and pulsate a light bulb. A key stepping stone toward full motor drive control on the Scrap Rover."
---

## Overview

Before committing to a full H-bridge motor driver circuit, I wanted to validate the fundamental building block: can the Raspberry Pi 5's 3.3 V GPIO reliably switch a load through a MOSFET using PWM? This milestone documents that proof-of-concept — a simple circuit where a light bulb pulsates under software control from the Pi.

It's a humble demo, but it proves the critical path: **GPIO → gate drive → MOSFET switching → real load**.

---

## The Circuit

The setup is straightforward:

- **MOSFET:** N-channel, logic-level compatible (gate threshold well within the Pi's 3.3 V GPIO output range)
- **Load:** A light bulb connected between the drain and the supply rail
- **Gate resistor:** ~330 Ω in series with the GPIO pin to limit inrush current and dampen ringing
- **Gate pull-down resistor:** ~10 kΩ from gate to GND to ensure the MOSFET stays off when the GPIO floats (e.g. during boot)
- **Flyback / freewheeling diode:** Across the load to clamp any inductive kickback (good practice even for resistive loads)
- **Power supply:** Separate from the Pi's 5 V rail — the Pi only sources the gate signal, not the load current

The gate resistor and pull-down form a simple voltage divider at low frequencies, but at PWM switching speeds the gate resistor mainly limits peak gate current. For a logic-level MOSFET driven at 3.3 V, Vgs at the gate will still be high enough for full enhancement-mode conduction.

---

## The Software

PWM is generated in Python using the `RPi.GPIO` library's software PWM. The script ramps the duty cycle up and down in a loop, creating the pulsating effect visible in the demo:

```python
from gpiozero import PWMLED
from time import sleep

led = PWMLED(17)

def pulsate(interval):
    brightness = 0.0
    while brightness <= 1.0:
        led.value = min(brightness, 1.0)
        brightness += interval
        sleep(0.05);

while True:
    pulsate(0.1);
```

> **Note:** Software PWM on the Pi has some jitter due to OS scheduling. For motor control, hardware PWM (available on GPIO 12, 13, 18, 19 in BCM) will give smoother results. For this visual demo it's perfectly adequate.

---

## What It Validates

This POC confirms several things needed for the upcoming H-bridge milestone:

| Question | Answer |
|---|---|
| Can 3.3 V GPIO fully enhance an N-channel logic-level MOSFET? | ✅ Yes |
| Does the gate resistor + pull-down behave as expected? | ✅ Yes |
| Can the Pi generate PWM fast enough for smooth control? | ✅ Yes (with software PWM; hardware PWM even better) |
| Does the MOSFET switch cleanly without thermal issues at low duty cycles? | ✅ Yes |

---

## Demo

{{< youtube D5gw2h_EEu4 >}}

---

## What's Next

Trying to hook the system up with only a powerbank and getting the light to work with higher voltages. 
I also want a button on the website to enable and disable the light pulsating to make a step in the direction 
of controlling the system with a controller and instead of a light using a motor. 
If it works with the controller I can normally control the motor speed with the joystick of the controller. 

