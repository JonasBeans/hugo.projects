---
title: "Milestone: Flask Server Integration — Web-Controlled LED Pulsating"
date: 2026-04-27
draft: false
tags: ["raspberry-pi", "flask", "python", "gpio", "pwm", "threading", "web"]
summary: "Integrating the PWM light control into the Scrap Rover's Flask server: a /switch/light endpoint toggles a smooth pulsating effect on and off, running in a background thread alongside camera PTZ and static file serving."
---

## Overview

With the MOSFET PWM circuit validated in the previous milestone, the next step was wiring the light control into the Scrap Rover's web server. The goal: a single HTTP call from the browser toggles the pulsating LED on and off — a direct stepping stone toward joystick-driven motor speed control.

This milestone covers integrating `gpiozero`'s `PWMLED` into a Flask app, running the pulsate loop in a background thread, and exposing a clean toggle endpoint.

---

## The Challenge: Blocking Loops in a Web Server

The original standalone pulsate script used a `while True` loop directly in the main thread:

```python
while True:
    pulsate(0.1)
```

Dropped into a Flask app, this blocks the server entirely — `app.run()` is never reached. The fix is running the loop in a daemon thread so Flask can start normally and the LED pulses independently.

A second problem: the original `pulsate` function only ramped brightness *up*. The new version does a full breathe cycle — up and back down — for a smoother visual effect.

---

## Threading Design

Two concerns drive the thread design:

1. **Stopping cleanly mid-ramp** — if the user toggles the light off while brightness is ramping up, the thread needs to exit promptly, not wait until the loop finishes.
2. **Avoiding stale threads** — calling `start_pulsating()` twice shouldn't spawn two competing threads.

Both are handled with a `threading.Event`:

```python
stop_pulsate = threading.Event()

def pulsate_loop():
    while not stop_pulsate.is_set():
        for i in range(0, 101, 5):
            if stop_pulsate.is_set():
                return
            led.value = i / 100
            sleep(0.03)
        for i in range(100, -1, -5):
            if stop_pulsate.is_set():
                return
            led.value = i / 100
            sleep(0.03)

def start_pulsating():
    global led_thread
    stop_pulsate.clear()
    led_thread = threading.Thread(target=pulsate_loop, daemon=True)
    led_thread.start()

def stop_pulsating():
    stop_pulsate.set()
    if led_thread:
        led_thread.join(timeout=1)
    led.value = 0
```

The event is checked at every brightness step, so the thread exits within one step (~30 ms) of being told to stop.

---

## The Toggle Endpoint

```python
led_enabled = True

@app.route('/switch/light', methods=['POST'])
def switch_light():
    global led_enabled
    led_enabled = not led_enabled
    if led_enabled:
        start_pulsating()
    else:
        stop_pulsating()
    return jsonify({"light": "on" if led_enabled else "off"})
```

A `POST` to `/switch/light` flips the state and returns the new value as JSON. The frontend just needs:

```javascript
fetch('/switch/light', { method: 'POST' });
```

---

## Full Server

The complete server brings together LED control, camera PTZ queuing, and static file serving:

```python
from flask import Flask, request, send_from_directory, jsonify
from gpiozero import PWMLED
from time import sleep
import requests
import threading
import queue

led = PWMLED(17)
app = Flask(__name__)

led_enabled = True
led_thread = None
stop_pulsate = threading.Event()

def pulsate_loop():
    while not stop_pulsate.is_set():
        for i in range(0, 101, 5):
            if stop_pulsate.is_set():
                return
            led.value = i / 100
            sleep(0.03)
        for i in range(100, -1, -5):
            if stop_pulsate.is_set():
                return
            led.value = i / 100
            sleep(0.03)

def start_pulsating():
    global led_thread
    stop_pulsate.clear()
    led_thread = threading.Thread(target=pulsate_loop, daemon=True)
    led_thread.start()

def stop_pulsating():
    stop_pulsate.set()
    if led_thread:
        led_thread.join(timeout=1)
    led.value = 0

start_pulsating()

camera_queue = queue.Queue(maxsize=1)

def camera_worker():
    while True:
        job = camera_queue.get()
        try:
            requests.post(
                "http://192.168.0.53/camera-cgi/com/ptz.cgi",
                data=job["data"],
                headers={
                    "Authorization": "Basic YWRtaW46MTIzNA==",
                    "Accept-Encoding": "gzip, deflate, br"
                },
                timeout=3
            )
        except Exception as e:
            print(f"Camera command failed: {e}")
        finally:
            camera_queue.task_done()

worker = threading.Thread(target=camera_worker, daemon=True)
worker.start()

@app.route('/')
def index():
    return send_from_directory('website', 'index.html')

@app.route('/<path:path>')
def static_files(path):
    return send_from_directory('website', path)

@app.route('/ptz', methods=['POST'])
def ptz():
    try:
        camera_queue.put_nowait({"data": request.data})
        return jsonify({"status": "queued"})
    except queue.Full:
        return jsonify({"status": "busy"}), 429

@app.route('/switch/light', methods=['POST'])
def switch_light():
    global led_enabled
    led_enabled = not led_enabled
    if led_enabled:
        start_pulsating()
    else:
        stop_pulsating()
    return jsonify({"light": "on" if led_enabled else "off"})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

---

## Running on Startup

The server is managed by a systemd service so it starts automatically on boot:

```ini
[Unit]
Description=Scrap Rover Python Server
After=network.target

[Service]
ExecStart=/usr/bin/python3 /home/scraprover/projects/scrap_rover/driver/server.py
WorkingDirectory=/home/scraprover/projects/scrap_rover/driver
Restart=on-failure
RestartSec=5
User=scraprover

[Install]
WantedBy=multi-user.target
```

One pitfall encountered: if the server crashes or is killed with `SIGKILL`, the GPIO pin can remain claimed by the kernel, causing a `GPIO busy` error on the next start. The `RestartSec=5` delay gives the kernel time to release it. For a clean shutdown, `led.close()` in a `SIGTERM` handler releases the claim explicitly.

---

## What It Validates

| Question | Answer |
|---|---|
| Can a blocking PWM loop coexist with a Flask server? | ✅ Yes, via daemon thread |
| Does `threading.Event` stop the loop cleanly mid-ramp? | ✅ Yes, exits within ~30 ms |
| Can the browser toggle the light with a single HTTP call? | ✅ Yes |
| Does the server start automatically on boot? | ✅ Yes, via systemd |

---

## Demo

{{< youtube 7Ia53AdONwQ >}}

## What's Next

With a working toggle endpoint, the next step is wiring a joystick input on the frontend to the same pattern — sending control commands over HTTP or WebSocket and mapping joystick axes to PWM duty cycles. Replacing the light with a motor and an H-bridge is then a hardware swap, not a software change.
