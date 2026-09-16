<p align="center"><img src="logo.png" alt="VisioNext" width="520"></p>

<p align="center">A camera-agnostic vision AI engine by <b>Inference Tech Sdn. Bhd.</b><br>Point it at any RTSP/RTMP camera or video file and receive detections as they happen.</p>

## About This Release

This is a basic demonstration release. Its purpose is to show the integration pathway: how VisioNext is deployed next to your cameras, how your application connects to it, and what it receives back. The two capabilities included, object detection and face recognition, are examples chosen to exercise that pathway end to end. They are not the extent of what VisioNext does. Further capabilities are delivered the same way, as additional containers speaking the same SDK, so an integration built against this release carries over unchanged.

Only these demonstration images are public. In production, every VisioNext image is private, pulled with credentials we issue to you, and covered by a commercial licensing agreement.

## How It Works

Each capability is a GPU container, published on Docker Hub under the `inferencetech` account. You run the containers with Docker Compose next to your cameras and talk to them with one Python SDK: add video sources, receive one record per sampled frame with the detections and the frame as JPEG. Nothing is written to disk. Your application decides what to keep.

This release ships two containers:

| Image | What it does | Events |
| --- | --- | --- |
| `inferencetech/visionext-objects` | Detects people and animals (bird, cat, dog, horse, sheep, cow, elephant, bear, zebra, giraffe) | `object.detected` |
| `inferencetech/visionext-faces` | Detects faces and optionally matches them against a photo gallery you supply | `face.detected`, `face.recognized` |

**Contents:** [About This Release](#about-this-release) · [How It Works](#how-it-works) · [Requirements](#requirements) · [Quick Start](#quick-start) · [Sources](#sources) · [SDK](#sdk) ([Errors](#errors), [`connect()` Options](#connect-options), [`Record`](#record), [`Event`](#event), [`Source`](#source)) · [Face Gallery](#face-gallery) · [Troubleshooting](#troubleshooting)

## Requirements

- Linux host with an NVIDIA GPU, NVIDIA driver 525 or newer, Docker Compose v2 and the [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html). The containers are GPU-only.
- Disk: about 16 GB for `visionext-objects` and 9 GB for `visionext-faces` once pulled.
- Python 3.10+ on the machine that will consume the records (the same host, or any machine that can reach the published ports).

## Quick Start

### 1. Start the Containers

Create a `compose.yaml`:

```yaml
services:
  objects:
    image: inferencetech/visionext-objects:1.0.0
    restart: unless-stopped
    gpus: all
    ports:
      - "127.0.0.1:8765:8765"
  faces:
    image: inferencetech/visionext-faces:1.0.0
    restart: unless-stopped
    gpus: all
    ports:
      - "127.0.0.1:8766:8765"
    volumes:
      - ./gallery:/gallery:ro     # optional, see Face gallery
```

```bash
docker compose up -d
```

The first start downloads the images (about 9 GB). Use port 8766 for faces. Pin a version tag in anything you deploy. The `gallery` volume is explained in [Face Gallery](#face-gallery).

### 2. Install the SDK

The SDK is a single Python wheel file, published in this repository next to this guide. Install the one whose version matches your image tag (in a virtualenv if you like):

```bash
pip install https://github.com/inference-asia/visionext/raw/main/visionext-1.0.0-py3-none-any.whl
```

That is all. The package is called `visionext` and pulls in its one dependency (`websockets`) by itself.

### 3. Add a Source and Read Records

```python
import visionext

stream = visionext.connect("ws://localhost:8765")
stream.add_source("rtsp://user:pass@camera-host:554/stream", source_id="lobby", every_n_frames=25)

for record in stream:
    for event in record.events:
        print(record.source_id, record.frame_index, event.label, round(event.confidence, 2), event.bbox.rounded())
```

Run it, and every 25th frame from the camera is printed with its detections. Ctrl-C stops it. The container keeps its sources until you remove them or it restarts. Everything the SDK offers is in [SDK](#sdk), and the fields of a record are in [`Record`](#record) and [`Event`](#event).

### 4. Objects and Faces at the Same Time

The two containers are independent. Each has its own endpoint (8765 for objects, 8766 for faces in the compose file above) and its own list of sources, so a source you want analysed by both must be added to both. Reading a stream blocks, so give each container a thread:

```python
import threading

import visionext

CAMERA = "rtsp://user:pass@camera-host:554/stream"


def watch(name, url):
    stream = visionext.connect(url)
    stream.add_source(CAMERA, source_id="lobby", every_n_frames=25)
    for record in stream:
        for event in record.events:
            print(name, record.source_id, record.frame_index, event.label, round(event.confidence, 2), flush=True)


threads = [
    threading.Thread(target=watch, args=("objects", "ws://localhost:8765"), daemon=True),
    threading.Thread(target=watch, args=("faces", "ws://localhost:8766"), daemon=True),
]
for t in threads:
    t.start()
for t in threads:
    t.join()
```

Without a gallery every face is reported as `unknown`. See [Face Gallery](#face-gallery) to get names.

## Sources

A source is a stream URL (`rtsp://`, `rtmp://`, anything FFmpeg can open) or a video file path **inside the container**. The container cannot see the host filesystem, so a host path such as `/home/me/video.mp4` is rejected. Mount the directory that holds your files into every container that should read them:

```yaml
    volumes:
      - ./videos:/input:ro
```

```python
stream.add_source("/input/clip.mp4", every_n_frames=10)
```

- `source_id` is carried by every record from the source. It defaults to the last path segment of the input without extension (`/input/clip.mp4` → `clip`). Letters, digits, `.`, `_` and `-` only, unique per container.
- `every_n_frames` processes every Nth frame of that source and drops the others.
- The container only processes frames while at least one client is connected. A file pauses while nobody is connected, so nothing in it is skipped.
- A file is read once. When it ends the container removes it and notifies your client (`ended`).
- A stream that drops is retried forever with exponential backoff (1 s doubling up to 60 s) until you remove it. Your client is notified when it is `lost` and when it is `opened` again. Adding a stream that is unreachable succeeds with status `reconnecting`.
- Sources live in the container's memory. After a restart the list is empty. The SDK re-adds the sources it added when it reconnects.
- Anyone who can reach the port can add sources and receive frames. Keep it on a private interface, as in the quick start, or behind an authenticating reverse proxy.

## SDK

```python
stream = visionext.connect("ws://localhost:8765")
```

`connect()` is lazy. It waits for the container to come up (the port opens once the model is loaded, a few seconds after start), reconnects if the container restarts, and re-adds the sources you added through it. Its arguments are listed under [`connect()` Options](#connect-options). Each record you iterate over is a [`Record`](#record) holding [`Event`](#event)s, and the source methods return [`Source`](#source) objects. Failures raise the exceptions in [Errors](#errors).

```python
stream.add_source(input, source_id=None, every_n_frames=1)   # returns a Source, see below
stream.remove_source(source_id)                               # returns the removed Source
stream.list_sources()                                         # list[Source]
stream.sources                                                # dict[source_id, Source], kept current
stream.close()                                                # stop iterating, from any thread or callback

for record in stream: ...                                     # every sampled frame
for record in stream.detections(): ...                        # only frames with detections
for event in stream.events(): ...                             # detections, each with event.record
```

Notifications about sources (`added`, `removed`, `opened`, `lost`, `ended`):

```python
def on_source(event, source):
    print(event, source.source_id, source.status)
    if event == "ended":
        stream.close()

stream = visionext.connect("ws://localhost:8765", on_source=on_source)
```

Keep only frames with detections and save their JPEGs:

```python
for record in stream.detections():
    record.frame.save(f"hits/{record.source_id}/{record.frame_index:012d}.jpg")
```

asyncio:

```python
async with visionext.aconnect("ws://localhost:8765") as stream:
    await stream.add_source("rtsp://…", every_n_frames=25)
    async for record in stream:
        ...
```

### Errors

```python
try:
    stream.add_source("/input/missing.mp4")
except visionext.RequestError as exc:
    print(exc.action, exc)        # add cannot open /input/missing.mp4
```

| Exception | When |
| --- | --- |
| `RequestError` | The container rejected the request: unreadable file (see [Troubleshooting](#troubleshooting)), `source_id` already used by another input, unknown `source_id`. `str(exc)` is the reason. |
| `RequestTimeout` | No reply within `request_timeout` (default 30 s). Adding an unreachable stream takes up to 10 s. |
| `StreamClosed` | The stream was closed while a request was pending. |
| `ConnectionError` | Nothing reachable within `connect_timeout` (default: wait forever). |

Stream drops and file ends are not exceptions. They arrive through `on_source` (see [Sources](#sources) for the events).

### `connect()` Options

| Argument | Default | Meaning |
| --- | --- | --- |
| `reconnect` | `True` | Reconnect after a container restart or network drop and re-add this stream's sources. |
| `retry_every` | `2.0` | Seconds between attempts while the container is unreachable. |
| `connect_timeout` | `None` | Raise `ConnectionError` after this many seconds without a connection. |
| `request_timeout` | `30.0` | Seconds to wait for a reply to a source request. |
| `on_source` | `None` | `callback(event, source)`. |

### `Record`

| Field | Meaning |
| --- | --- |
| `source_id` | The source the frame came from. |
| `timestamp` | Processing time, timezone-aware `datetime` (UTC). |
| `frame_index` | 1-based frame number within the source, counting every frame read. With `every_n_frames=25` you get 25, 50, 75, … |
| `frame_width`, `frame_height`, `size` | Pixel size. |
| `events` | `list[Event]`, see [`Event`](#event). `objects` and `faces` filter by kind. Empty when nothing was detected. |
| `frame` | The unannotated frame as JPEG: `frame.bytes`, `frame.save(path)`, `frame.to_pil()` (needs Pillow), `frame.to_numpy()` (needs numpy and OpenCV, BGR). |

### `Event`

| Field | Meaning |
| --- | --- |
| `name` | `object.detected`, `face.detected` or `face.recognized`. Shortcuts: `is_object`, `is_face`, `recognized`. |
| `label` | Readable: the class name (`"person"`, `"dog"`, `"horse"`) for objects, the gallery identity or `"unknown"` for faces. |
| `confidence` | 0 to 1. Only detections at or above the detector's threshold (0.5) are emitted. |
| `bbox` | `BBox(x1, y1, x2, y2)` in pixels, with `width`, `height`, `area`, `center`, `rounded()`, `contains(x, y)`. |
| `class_id` | Objects: numeric class id. `1` is person, `16` to `25` are the animals. `visionext.COCO_LABELS` maps ids to names. |
| `identity`, `similarity`, `landmarks` | Faces: gallery identity (see [Face Gallery](#face-gallery)), cosine similarity to it, five `Point(x, y)` (eyes, nose, mouth corners). |
| `record` | The `Record` this event belongs to. |

### `Source`

`source_id`, `input` (credentials removed), `kind` (`file` or `stream`), `every_n_frames`, `status` (`open`, `reconnecting`, `ended`, `removed`), `frame_index`.

## Face Gallery

`face-detector` recognizes people from photos you provide. One sub-folder per person, named with the identity you want in `event.identity`, mounted read-only at `/gallery`:

```
gallery/
  alice/   front.jpg  side.jpg
  bob/     bob.png
```

- Formats: `.jpg .jpeg .png .bmp .webp`. Clear, frontal faces, 2 to 5 photos per person. Frames from the camera itself are fine.
- Read once at startup. `docker compose restart faces` after changing photos. `docker compose logs faces` shows `Gallery: N identities from M images`.
- No gallery means detection only. Every face is `"unknown"` and `event.recognized` is false.
- A match needs similarity 0.45 or more. If someone is found but reported unknown with similarity just below that, add more clear photos of them.

Face photos are biometric data. Make sure you have consent and a legal basis before deploying recognition.

## Troubleshooting

| Symptom | Cause / fix |
| --- | --- |
| `CUDA is required but unavailable` in `docker compose logs` | The GPU is not visible to the container. Check `gpus: all`, the NVIDIA Container Toolkit, and that `nvidia-smi` works on the host. |
| `connect()` keeps waiting | The container is still loading (`docker compose logs` shows `Listening on` when ready), the port is not published, or the host port is wrong. |
| Connected but no records | No sources (`stream.list_sources()`), or the source is `reconnecting`: check the URL and credentials. See [Sources](#sources). |
| `RequestError: no such file inside the container: …` | You passed a host path, or the volume is not mounted. Mount the directory (see [Sources](#sources)) and use the path inside the container, for example `/input/video.mp4`. |
| `Client … is not keeping up: dropping its oldest records` in the logs | Your client reads slower than records are produced. Read faster or raise `every_n_frames`. |
| Every face is `unknown` despite a gallery | Check the logs for `Gallery: N identities`. 0 means the folder is not mounted at `/gallery` or has no readable faces. See [Face Gallery](#face-gallery). |
