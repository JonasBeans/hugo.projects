+++
title = 'Streaming the IP camera with WebRTC'
date = 2026-03-29
draft = false
tags = ['raspberry-pi', 'webrtc', 'rtsp', 'mediamtx', 'networking']
+++

## Introduction

Now that the IP camera is confirmed working over RTSP, the next challenge is getting that stream
into a browser without plugins or heavy client software. The solution: convert the RTSP stream to
**WebRTC** and serve it directly from the Pi's own webpage.

The full pipeline looks like this:

```
IP Camera (RTSP) → MediaMTX → WebRTC → Browser
```

WebRTC is natively supported in all modern browsers and brings latency down to
**under 200ms** — a significant improvement over the ~500ms delay observed in VLC.

---

## Why WebRTC?

RTSP is great for local network streaming tools like VLC, but browsers cannot consume it natively.
Options like HLS or DASH work in browsers but introduce multi-second latency. WebRTC is the sweet
spot: browser-native, low latency, and no plugins required.

---

## The tool: MediaMTX

[MediaMTX](https://github.com/bluenviron/mediamtx) (formerly rtsp-simple-server) is the bridge
between RTSP and WebRTC. It:

- Runs on Raspberry Pi (ARM binary available)
- Accepts an RTSP source and re-publishes it as WebRTC automatically
- Handles all WebRTC signaling internally via the **WHEP** protocol
- Ships as a single binary — no complex dependencies

If the camera outputs **H.264**, MediaMTX can pass the stream through to the browser without
transcoding, keeping CPU usage low on the Pi.

---

## Setting up MediaMTX

### 1. Download and extract

```bash
# Use arm64v8 for 64-bit Pi OS, or armv7 for 32-bit
wget https://github.com/bluenviron/mediamtx/releases/latest/download/mediamtx_linux_arm64v8.tar.gz
tar -xzf mediamtx_linux_arm64v8.tar.gz
```

### 2. Configure the stream source

Edit `mediamtx.yml` and point it at the camera's RTSP URL:

```yaml
paths:
  cam:
    source: rtsp://admin:password@192.168.1.100:554/stream
```

That's all the configuration needed. MediaMTX handles the rest automatically.

### 3. Run it

```bash
./mediamtx
```

MediaMTX will start pulling the RTSP stream and expose it as a WebRTC endpoint at:

```
http://<pi-ip>:8889/cam/whep
```

---

## Autostart on boot with systemd

To have MediaMTX start automatically when the Pi boots:

```bash
sudo nano /etc/systemd/system/mediamtx.service
```

```ini
[Unit]
Description=MediaMTX
After=network.target

[Service]
ExecStart=/home/pi/mediamtx/mediamtx /home/pi/mediamtx/mediamtx.yml
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable mediamtx
sudo systemctl start mediamtx
```

---

## Embedding the stream in the webpage

With MediaMTX running, embedding the live feed is just a `<video>` element and a small
JavaScript WHEP client.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Scrap Rover</title>
  <style>
    body {
      margin: 0;
      background: #111;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      min-height: 100vh;
      font-family: sans-serif;
      color: white;
    }
    video {
      width: 100%;
      max-width: 800px;
      border-radius: 8px;
      background: #000;
    }
  </style>
</head>
<body>
  <h2>🛰 Scrap Rover Live Feed</h2>
  <video id="stream" autoplay muted playsinline controls></video>

  <script>
    async function startStream() {
      const pc = new RTCPeerConnection();

      // When a video track arrives, attach it to the <video> element
      pc.ontrack = (event) => {
        document.getElementById('stream').srcObject = event.streams[0];
      };

      pc.addTransceiver('video', { direction: 'recvonly' });
      pc.addTransceiver('audio', { direction: 'recvonly' });

      // Create WebRTC offer
      const offer = await pc.createOffer();
      await pc.setLocalDescription(offer);

      // Send offer to MediaMTX WHEP endpoint and get answer
      const response = await fetch('http://localhost:8889/cam/whep', {
        method: 'POST',
        headers: { 'Content-Type': 'application/sdp' },
        body: offer.sdp,
      });

      const answerSdp = await response.text();
      await pc.setRemoteDescription({ type: 'answer', sdp: answerSdp });
    }

    startStream();
  </script>
</body>
</html>
```

> **Note:** The fetch URL uses `localhost` — this works if the browser runs on the Pi itself.
> When connecting from a phone or laptop on the Pi's WiFi network, replace `localhost` with
> the Pi's IP address (e.g. `192.168.4.1` if the Pi is acting as an access point).

---

## How it all fits together

Once everything is running, the full flow is:

1. Pi boots → **MediaMTX starts** and pulls the RTSP stream from the IP camera
2. A device connects to the **Pi's WiFi network**
3. The user opens the **webpage** served by the Pi
4. JavaScript negotiates a WebRTC connection with MediaMTX via WHEP
5. The live video feed appears in the browser in **under 200ms**

---

## Up next

With a working live feed in the browser, the next step is adding the control interface —
buttons or a joystick on the same page that send commands to the Pi's GPIO pins to drive
the motors forward, backward, and eventually steer.
