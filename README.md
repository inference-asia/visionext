<p align="center"><img src="logo.png" alt="VisioNext" width="520"></p>

<p align="center">A camera-agnostic vision AI engine by <b>Inference Tech Sdn. Bhd.</b><br>Connect any RTSP/RTMP camera or video file and receive detections as they happen.</p>

## About This Release

This is a **demonstration release**. It exists to show the **integration pathway**: how VisioNext runs next to your cameras, how your application connects to it, and what it gets back.

It ships with two capabilities, object detection and face recognition. **They are examples, not the product.** VisioNext delivers every capability the same way, as a container that speaks the same SDK, so **whatever you build against this release keeps working** as more capabilities are added.

## How It Works

Every capability runs as a GPU container, published on Docker Hub under the `inferencetech` account. You start the containers with Docker Compose on a machine that can reach your cameras, then drive them from your application with the Python SDK: add a video source, and for every sampled frame you receive the detections together with the frame as a JPEG. **Nothing is written to disk.** Your application decides what to keep.

This release includes:

| Image | What it does | Events |
| --- | --- | --- |
| `inferencetech/visionext-objects` | Detects people and animals (bird, cat, dog, horse, sheep, cow, elephant, bear, zebra, giraffe) | `object.detected` |
| `inferencetech/visionext-faces` | Detects faces and optionally matches them against a photo gallery you supply | `face.detected`, `face.recognized` |

A third image, `inferencetech/visionext-dashboard`, is a web page for operating the engines by hand: add sources, watch the frames and their detections, inspect records. It needs no GPU and no code. See [Dashboard](#dashboard).

These demonstration images are public. **Production images are private**, pulled with credentials we issue, and covered by a **commercial licensing agreement**.

**Contents:** [About This Release](#about-this-release) · [How It Works](#how-it-works) · [Requirements](#requirements) · [Quick Start](#quick-start) · [Dashboard](#dashboard) · [Sources](#sources) · [SDK](#sdk) ([Errors](#errors), [`connect()` Options](#connect-options), [`Record`](#record), [`Event`](#event), [`Source`](#source)) · [Face Gallery](#face-gallery) · [Troubleshooting](#troubleshooting)

## Requirements

- Linux host with an NVIDIA GPU, NVIDIA driver 525 or newer, Docker Compose v2 and the [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html). The containers are GPU-only.
- Disk: about 16 GB for `visionext-objects` and 9 GB for `visionext-faces` once pulled. The dashboard is under 100 MB.
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
  dashboard:
    image: inferencetech/visionext-dashboard:1.0.0
    restart: unless-stopped
    ports:
      - "127.0.0.1:8080:80"
```

```bash
docker compose up -d
```

The first start downloads the images (about 9 GB). Use port 8766 for faces. Pin a version tag in anything you deploy. The `gallery` volume is explained in [Face Gallery](#face-gallery). Keep the service names `objects` and `faces` as written, the dashboard finds the engines by those names.

Open [http://localhost:8080](http://localhost:8080) to see the engines come up. Each one shows **Connected** once its model is loaded, which takes a few seconds after start. You can add a camera there right away, see [Dashboard](#dashboard), or continue with the SDK.

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

## Dashboard

The dashboard is the same client as your application, in a browser. One tab per engine, each with:

- **Sources**: add a stream URL or a file path inside the container, set `every_n_frames`, remove sources. The list is the engine's own, so sources added from the SDK appear here and the other way round.
- **Live**: the latest sampled frame of every source with its detections drawn on it. Click a frame to see the record behind it, its events, and the JSON your application receives. Pause freezes the view while the counters keep running.
- **Records** and **Detections**: the most recent records and events, newest first.

Two things to know:

- The dashboard is a client. While a tab is open the engines process frames for it, so a file source is read while you watch even if no application is connected.
- It has no login. Anyone who can open the page can add sources and see frames. Publish it on a private interface, as in the quick start, or behind an authenticating reverse proxy, and never on the public internet.

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
- The container **only processes frames while at least one client is connected**. A file pauses while nobody is connected, so nothing in it is skipped.
- A file is read once. When it ends the container removes it and notifies your client (`ended`).
- A stream that drops is retried forever with exponential backoff (1 s doubling up to 60 s) until you remove it. Your client is notified when it is `lost` and when it is `opened` again. Adding a stream that is unreachable succeeds with status `reconnecting`.
- Sources live in the container's memory. **After a restart the list is empty.** The SDK re-adds the sources it added when it reconnects.
- **Anyone who can reach the port can add sources and receive frames.** Keep it on a private interface, as in the quick start, or behind an authenticating reverse proxy.

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

`visionext-faces` recognizes people from photos you provide. One sub-folder per person, named with the identity you want in `event.identity`, mounted read-only at `/gallery`:

```
gallery/
  alice/   front.jpg  side.jpg
  bob/     bob.png
```

- Formats: `.jpg .jpeg .png .bmp .webp`. Clear, frontal faces, 2 to 5 photos per person. Frames from the camera itself are fine.
- Read once at startup. `docker compose restart faces` after changing photos. `docker compose logs faces` shows `Gallery: N identities from M images`.
- No gallery means detection only. Every face is `"unknown"` and `event.recognized` is false.
- A match needs similarity 0.45 or more. If someone is found but reported unknown with similarity just below that, add more clear photos of them.

Face photos are biometric data. **Make sure you have consent and a legal basis** before deploying recognition.

## Troubleshooting

| Symptom | Cause / fix |
| --- | --- |
| `CUDA is required but unavailable` in `docker compose logs` | The GPU is not visible to the container. Check `gpus: all`, the NVIDIA Container Toolkit, and that `nvidia-smi` works on the host. |
| The dashboard shows **Reconnecting** for an engine | The engine is still loading (a few seconds after start), or its compose service is not named `objects` / `faces`. |
| `connect()` keeps waiting | The container is still loading (`docker compose logs` shows `Listening on` when ready), the port is not published, or the host port is wrong. |
| Connected but no records | No sources (`stream.list_sources()`), or the source is `reconnecting`: check the URL and credentials. See [Sources](#sources). |
| `RequestError: no such file inside the container: …` | You passed a host path, or the volume is not mounted. Mount the directory (see [Sources](#sources)) and use the path inside the container, for example `/input/video.mp4`. |
| `Client … is not keeping up: dropping its oldest records` in the logs | Your client reads slower than records are produced. Read faster or raise `every_n_frames`. |
| Every face is `unknown` despite a gallery | Check the logs for `Gallery: N identities`. 0 means the folder is not mounted at `/gallery` or has no readable faces. See [Face Gallery](#face-gallery). |
