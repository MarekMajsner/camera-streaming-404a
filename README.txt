Lab 404a RealSense Camera Streaming
===================================

Simple Docker Compose + MediaMTX setup for streaming Intel RealSense cameras
from Lab 404a over the local LAN or Tailscale.

Each camera is captured via V4L2 using FFmpeg inside the MediaMTX container,
encoded to low-latency H.264, and served over RTSP, WebRTC, and HLS.


Prerequisites
-------------
- Docker with the Compose plugin (`docker compose`)
- Intel RealSense cameras connected via USB
- (Optional) Tailscale running on the host and client for remote access
- (Optional) `v4l-utils` on the host if you need to inspect/add cameras:
    sudo apt install v4l-utils


Quick Start (on the Lab 404a host)
----------------------------------
Start streaming in the background:
    docker compose up -d

View logs (useful for verifying cameras started cleanly):
    docker compose logs -f

Apply changes after editing `mediamtx.yml`:
    docker compose restart

Stop streaming:
    docker compose down

Note: The container uses `restart: unless-stopped`, so streams will come back
automatically after a reboot.


Viewing the Streams
-------------------
Replace `<HOST>` with the Lab 404a machine's local LAN IP, Tailscale IP
(`100.x.x.x`), or Tailscale MagicDNS hostname.

Configured streams (see `mediamtx.yml`): `cam1`, `cam2` (`cam3`/`cam4` are
commented-out templates).

1. Web Browser (WebRTC - easiest, lowest latency)
   - http://<HOST>:8889/cam1
   - http://<HOST>:8889/cam2

2. Web Browser / Media Player (HLS - higher latency, wide compatibility)
   - http://<HOST>:8888/cam1
   - http://<HOST>:8888/cam2

3. RTSP (VLC, ffplay, mpv, OpenCV, etc.)
   Stream URLs:
   - rtsp://<HOST>:8554/cam1
   - rtsp://<HOST>:8554/cam2

   Examples:
   - ffplay (low latency, TCP):
       ffplay -fflags nobuffer -flags low_delay -rtsp_transport tcp rtsp://<HOST>:8554/cam1

   - mpv (low latency, TCP):
       mpv --profile=low-latency --rtsp-transport=tcp rtsp://<HOST>:8554/cam1

   - VLC:
       vlc --rtsp-tcp rtsp://<HOST>:8554/cam1
     (or open via Media -> Open Network Stream)


Networking & Ports
------------------
The container runs with `network_mode: host`, so it listens directly on both
LAN and Tailscale interfaces. `udpMaxPayloadSize: 1200` is set in `mediamtx.yml`
to avoid UDP packet fragmentation over Tailscale's MTU.

If a host firewall (e.g. `ufw`) is enabled, allow:
- 8554/tcp, 8554/udp : RTSP
- 8889/tcp, 8189/udp : WebRTC (HTTP signaling + ICE UDP)
- 8888/tcp           : HLS


Adding or Changing Cameras
--------------------------
Cameras are referenced by `/dev/v4l/by-path/...` so each stream stays pinned to
a specific physical USB port across reboots.

1. List connected video devices and their USB paths:
       ls -l /dev/v4l/by-path/
       v4l2-ctl --list-devices

2. RealSense cameras expose multiple `/dev/video*` nodes (depth, IR, RGB).
   Find the RGB node supporting `YUYV` (typically `...-1.3-video-index0` on
   RealSense D4xx cameras):
       v4l2-ctl -d /dev/v4l/by-path/<DEVICE> --list-formats-ext

3. Edit `mediamtx.yml`: uncomment `cam3` or copy the `cam4` template, set the
   `-i /dev/v4l/by-path/...` path, and match the stream name at the end of the
   `rtsp://127.0.0.1:8554/<name>` URL.
   (You can also adjust `-video_size` and `-framerate` here if supported by the
   camera.)

4. Restart MediaMTX:
       docker compose restart


Troubleshooting
---------------
- Camera not streaming after replugging:
  Plugging a camera into a different USB port changes its `/dev/v4l/by-path/`
  path. Either plug it back into the original port or update `mediamtx.yml`.

- "Device or resource busy" in logs:
  Another application (e.g. `realsense-viewer`, ROS, or a Python script) is
  currently using the camera. Stop that process or stop this container when
  using the cameras locally.

- Packet loss or artifacts over Tailscale:
  Use TCP transport for RTSP (`-rtsp_transport tcp` / `--rtsp-tcp`), or view
  via WebRTC (`http://<HOST>:8889/<cam>`).

- Check FFmpeg / MediaMTX errors:
      docker compose logs --tail=100 mediamtx
