## Installing Media MTX

To start out my journey on discovering how Media MTX works, I installed the binary on my Windows laptop.

It came as .zip file so after extracting this file I was presented with a .exe file.

Executing this binary resulted in a warning which I ignored and let it run anyway.

After this a new console window opened which gave the following logs:



```log

2026/03/30 16:27:40 INF MediaMTX v1.17.0, windows, amd64

2026/03/30 16:27:40 INF configuration loaded from C:\\\\Projecten\\\\Tools\\\\mediamtx\\\_v1.17.0\\\_windows\\\_amd64\\\\mediamtx.yml

2026/03/30 16:27:40 INF \\\[RTSP] listener opened on :8554 (TCP/RTSP), :8000 (UDP/RTP), :8001 (UDP/RTCP)

2026/03/30 16:27:40 INF \\\[RTMP] listener opened on :1935

2026/03/30 16:27:40 INF \\\[HLS] listener opened on :8888

2026/03/30 16:27:40 INF \\\[WebRTC] listener opened on :8889 (HTTP), :8189 (ICE/UDP)

2026/03/30 16:27:40 INF \\\[SRT] listener opened on :8890 (UDP)

``` 



My realization came that this will start some sort of server which can serve request. 
I haven't however found a way to configure the source of the video that should be streamed. 



===



*Update: * 
I managed to find a way to configure a the source. When unpacking the .zip file it comes with a .yml config file where 
a source path can be configured. It is possible to just insert a connection string like: 

`rtsp://\[user]\[password]@\[host]/\[location]`

Then when requesting the location in a browser on: `localhost:8889/scrap\_rover\_cam` I managed to get some audio. 
The video stream however doesn't work still. So back to the drawing board, figuring that out. 

=== 

*Update:*

Okay so a possible explanation for the problem is that WebRTC doesn't support MJPEG. 

A solution would be to use FFMPEG to convert the stream into H.264 (Whatever that is) and let WebRTC 
stream that instead. 


Let's try it!


=== 

*Update: *

So in the end it worked the architecture looks a bit like this: 
[IP camera] -> (RTSP, MJPEG) -> [FFMPEG] -> (RTSP, H.264) -> [MediaMTX] -> (WebRTC) -> [Browser]

An incoming video stream which is in the form of MJPEG has to be converted into H.264. 
This is so because WebRTC doesn't support MJPEG. The way we can do this is through a tool called: FFMPEG. 
Suposedly this a the God tier tool which is used every. It can basicly convert almost every kind of 
incoming source stream type into almost any available type. 
As a reference I used [this website](https://hackernoon.com/part-1building-your-first-video-pipeline-ffmpeg-and-mediamtx-basics).

And the website code looks like this: [GitHub repository](https://github.com/JonasBeans/scrap_rover.git)

I wanted to discover how to rotate the camera. With this type of camera it has this feature. 
To do so I opened up Burp Suite and intercepted the request that are made through the admin console. 
To conclude it's a basic webrequest actualy. 

```http
POST /camera-cgi/com/ptz.cgi HTTP/1.1
Host: 192.168.x.x
Content-Length: 10
Authorization: Basic {Insert credentials}
move=right
```

To make it move right. It's that easy. :)
