# Scrap Rover — Controls & Architecture Notes

## Overview

The Scrap Rover is a remotely controlled vehicle built from scrap parts: a Raspberry Pi, an IP camera, and wheels. The controller interface is a webpage served by the Pi itself, accessible via Wi-Fi.

---

## UI Layout

The webpage consists of two sections:

- **Video stream** — embedded as an HTML element at the top
- **Control buttons** — directional buttons below the stream

```
┌─────────────────────┐
│                     │
│    video stream     │
│                     │
├─────────────────────┤
│        [▲]          │
│    [◄] [▼] [►]      │
└─────────────────────┘
```

PC users can additionally use **arrow keys** for input.

---

## Input Handling

| Input method | Events used |
|---|---|
| Touch (mobile) | `touchstart` / `touchend` |
| Mouse (desktop) | `mousedown` / `mouseup` |
| Keyboard (desktop) | `keydown` / `keyup` |

All three methods call the same shared WebSocket send function.

> **Note:** Use `preventDefault()` on touch events to prevent the page from scrolling when pressing buttons. Buttons should be large touch targets on mobile.

---

## Communication — WebSockets

WebSockets are the chosen technology for sending control commands from the browser to the Pi.

**Why WebSockets:**
- Low latency, persistent connection
- No HTTP overhead per command
- Works natively in the browser
- Lightweight server possible with Python (`asyncio` + `websockets`)

**Message format:**
```json
{ "action": "forward", "state": "start" }
{ "action": "forward", "state": "stop" }
```

Commands are sent on press (`start`) and release (`stop`), enabling held-down movement.

### ⚠️ Watchdog Timer

If the Wi-Fi connection drops, the rover will keep its last state (e.g. keep driving forward).

**Solution:** implement a watchdog on the Pi that stops all motors if no message is received within ~500ms.

---

## Alternative Technologies Considered

| Technology | Verdict |
|---|---|
| **WebSockets** | ✅ Recommended — simple, low latency, browser-native |
| **MQTT** | ⚠️ Good alternative if Wi-Fi is unreliable — requires a broker (e.g. Mosquitto) |
| **WebRTC Data Channels** | ⚠️ Lower latency but significantly more complex to set up |
| **Bluetooth** | ❌ Requires pairing, breaks web interface concept, limited range |
| **Raw UDP** | ❌ Browsers cannot send raw UDP — requires a native app |

---

## Video Stream

RTSP (used by the IP camera) cannot be embedded directly in a browser. Two options:

| Option | How | Latency |
|---|---|---|
| **FFmpeg → MJPEG** | Pipe RTSP to MJPEG, serve as HTTP stream, embed with `<img src="...">` | Moderate |
| **go2rtc** | Restream RTSP as WebRTC into the browser | Low |

MJPEG is the simpler starting point. `go2rtc` is worth revisiting if latency becomes an issue.

---

## Stack Summary

| Component | Technology |
|---|---|
| Web server | Raspberry Pi (hosts files + WebSocket server) |
| Network | Pi serves its own Wi-Fi hotspot |
| Frontend | Plain HTML / CSS / JS (no framework needed) |
| Communication | WebSockets |
| GPIO control | Python (`RPi.GPIO` or `gpiozero`) |
| Video stream | FFmpeg → MJPEG (or go2rtc for lower latency) |
