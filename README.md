# Lab 404a RealSense Camera Streaming

Simple Docker Compose + [MediaMTX](https://github.com/bluenviron/mediamtx) setup for streaming Intel RealSense cameras from Lab 404a over the local LAN or Tailscale.

Each camera is captured via V4L2 using FFmpeg inside the MediaMTX container, encoded to low-latency H.264, and served over WebRTC.

## Prerequisites

- Docker with the Compose plugin (`docker compose`)
- Intel RealSense cameras connected via USB
- *(Optional)* [Tailscale](https://tailscale.com/) running on both host and client for remote access
- *(Optional)* `v4l-utils` on the host if you need to inspect or add cameras:
  
  ```bash
  sudo apt install v4l-utils
  ```

## Quick Start

Run these commands on the Lab 404a host from this repository directory:

```bash
# Start streaming in the background
docker compose up -d
```

### Useful commands

```bash
# View logs (useful for verifying cameras started cleanly)
docker compose logs -f

# Apply changes after editing mediamtx.yml
docker compose restart

# Stop streaming
docker compose down
```

> [!NOTE]
> The container uses `restart: unless-stopped`, so streams will resume automatically after a host reboot.

## Viewing the Streams

Replace `<HOST>` with the Lab 404a machine's local LAN IP, Tailscale IP (`100.x.x.x`), or Tailscale MagicDNS hostname.

Configured streams (see `mediamtx.yml`): `cam1`, `cam2` (`cam3` and `cam4` are commented-out templates).

For quick viewing in browser: `http://<HOST>:8889/cam1`

## Adding or Changing Cameras

Cameras are referenced by `/dev/v4l/by-path/...` so each stream remains pinned to a specific physical USB port across reboots.

1. List connected video devices and their USB paths:

   ```bash
   ls -l /dev/v4l/by-path/
   v4l2-ctl --list-devices
   ```

2. RealSense cameras expose multiple `/dev/video*` nodes (depth, IR, RGB). Find the RGB node supporting `YUYV` (typically `...-1.3-video-index0` on RealSense D4xx cameras):

   ```bash
   v4l2-ctl -d /dev/v4l/by-path/<DEVICE> --list-formats-ext
   ```

3. Edit `mediamtx.yml`: uncomment `cam3` or copy the `cam4` template, update the `-i /dev/v4l/by-path/...` input path, and match the stream name at the end of `rtsp://127.0.0.1:8554/<name>`. You can also adjust `-video_size` and `-framerate` if supported by the camera.

4. Restart MediaMTX:

   ```bash
   docker compose restart
   ```

## Troubleshooting

- **Inspect FFmpeg / MediaMTX errors:**

  ```bash
  docker compose logs --tail=100 mediamtx
  ```
