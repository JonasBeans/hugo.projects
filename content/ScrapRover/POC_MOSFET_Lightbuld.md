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
import RPi.GPIO as GPIO
import time

PWM_PIN = 18  # BCM numbering
FREQUENCY = 1000  # Hz

GPIO.setmode(GPIO.BCM)
GPIO.setup(PWM_PIN, GPIO.OUT)

pwm = GPIO.PWM(PWM_PIN, FREQUENCY)
pwm.start(0)

try:
    while True:
        # Ramp up
        for dc in range(0, 101, 5):
            pwm.ChangeDutyCycle(dc)
            time.sleep(0.05)
        # Ramp down
        for dc in range(100, -1, -5):
            pwm.ChangeDutyCycle(dc)
            time.sleep(0.05)
except KeyboardInterrupt:
    pass
finally:
    pwm.stop()
    GPIO.cleanup()
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

This single low-side switch is only half the story. To drive a motor in **both directions**, I need an H-bridge — four switches arranged so current can be sent through the motor in either polarity.

The plan:
- **4-MOSFET discrete H-bridge** using logic-level N-channel and P-channel MOSFETs, or
- **Darlington fallback** with TIP120/TIP125 pairs using available salvaged parts (with the trade-off of ~1–2 V Vce(sat) drop at the transistor)
- Or a dedicated IC like the **DRV8833** or **TB6612FNG** for a cleaner, integrated solution

The MOSFET POC here gives me confidence in the gate drive side. Next up: direction switching.