<p align="center"><img src="logo.png" alt="VisioNext" width="520"></p>

<p align="center">Containerised vision inference for RTSP/RTMP cameras and video files, by <b>Inference Tech Sdn. Bhd.</b></p>

## Overview

VisioNext packages each vision capability as a GPU container that ingests video sources and delivers structured detections to connected clients. Deployment is a Docker Compose file, integration is a Python SDK, and day-to-day operation is a browser dashboard. Every sampled frame produces one record holding the detections and the frame itself as a JPEG. The engine retains nothing on disk. What to keep, and for how long, is the consumer's decision.

This demonstration release contains two engines. They illustrate the integration model rather than define the product: each VisioNext capability is delivered as a container with an identical interface, so code written against this release applies unchanged to capabilities released later.

| Image | Function | Events |
| --- | --- | --- |
| `inferencetech/visionext-objects` | Detection of people and animals (bird, cat, dog, horse, sheep, cow, elephant, bear, zebra, giraffe) | `object.detected` |
| `inferencetech/visionext-faces` | Face detection, with identification against a photo gallery when one is supplied | `face.detected`, `face.recognized` |

A third image, `inferencetech/visionext-dashboard`, provides a browser interface for operating the engines without code. It requires no GPU. See [Dashboard](#dashboard).

The demonstration images are public on Docker Hub. Production images are distributed privately under a commercial licensing agreement, with pull credentials issued by Inference Tech.

**Contents:** [Requirements](#requirements) · [Deployment](#deployment) · [Integration](#integration) · [Sources](#sources) · [SDK Reference](#sdk-reference) · [Dashboard](#dashboard) · [Face Gallery](#face-gallery) · [Security](#security) · [Troubleshooting](#troubleshooting)

## Requirements

- A Linux host with an NVIDIA GPU, NVIDIA driver 525 or later, Docker Compose v2 and the [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html). The engines do not run on CPU.
- Disk space of roughly 16 GB for `visionext-objects` and 9 GB for `visionext-faces`. The dashboard is under 100 MB.
- Python 3.10 or later on any machine with network access to the published ports.

## Deployment

Place the following in `compose.yaml` and run `docker compose up -d`:

```yaml
services:
  objects:
    image: inferencetech/visionext-objects:1.0.0
    restart: unless-stopped
    gpus: all
    ports:
      - "127.0.0.1:8765:8765"
    volumes:
      - uploads:/uploads:ro       # videos uploaded through the dashboard
  faces:
    image: inferencetech/visionext-faces:1.0.0
    restart: unless-stopped
    gpus: all
    ports:
      - "127.0.0.1:8766:8765"
    volumes:
      - uploads:/uploads:ro
      - ./gallery:/gallery:ro     # optional, see Face Gallery
  dashboard:
    image: inferencetech/visionext-dashboard:1.0.0
    restart: unless-stopped
    ports:
      - "8767:80"
    volumes:
      - uploads:/uploads

volumes:
  uploads:
```

The first start pulls about 9 GB. The service names `objects` and `faces` are significant, since the dashboard resolves the engines by name. Pin an explicit version tag in anything you deploy.

An engine accepts connections once its model is loaded, a few seconds after start. The dashboard at `http://<host>:8767` reports each engine as **Connected** at that point. The compose file binds the engines to the loopback interface and the dashboard to all interfaces. See [Security](#security) before changing either.

### SDK

The SDK is distributed as a wheel in this repository. Its version tracks the image tag, so install the wheel that matches the images you deployed:

```bash
pip install https://github.com/inference-asia/visionext/raw/main/visionext-1.0.0-py3-none-any.whl
```

The package imports as `visionext` and depends only on `websockets`.

## Integration

### Consuming Records

```python
import visionext

stream = visionext.connect("ws://localhost:8765")
stream.add_source("rtsp://user:pass@camera-host:554/stream", source_id="lobby", every_n_frames=25)

for record in stream:
    for event in record.events:
        print(record.source_id, record.frame_index, event.label, round(event.confidence, 2), event.bbox.rounded())
```

Iteration blocks and yields one [`Record`](#record) for every sampled frame from every source registered with the engine. Each record carries its [`Event`](#event)s. Ctrl-C terminates the script. The engine keeps the source until it is removed or the engine restarts.

### Running Both Engines

The engines are independent, each with its own endpoint (8765 and 8766 in the compose file above) and its own source list. A source that needs both analyses is added to both. Because iteration blocks, dedicate a thread to each engine:

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

Without a gallery, face events carry the identity `unknown`. See [Face Gallery](#face-gallery).

## Sources

A source is either a stream URL (`rtsp://`, `rtmp://`, or any scheme FFmpeg supports) or the path of a video file inside the container. The engine has no view of the host filesystem, so files reach it only through a mounted volume:

```yaml
    volumes:
      - ./videos:/input:ro
```

```python
stream.add_source("/input/clip.mp4", every_n_frames=10)
```

Videos uploaded through the [Dashboard](#dashboard) are mounted in every engine at `/uploads/<name>`.

### Identification and Sampling

- `source_id` labels every record from the source. It defaults to the file name without extension (`/input/clip.mp4` becomes `clip`). Permitted characters are letters, digits, `.`, `_` and `-`, and ids are unique per engine.
- `every_n_frames` sets the sampling interval. Frames N, 2N, 3N and so on are analysed and the remainder are discarded.

### Lifecycle

- Analysis is gated on client presence. With no client connected, streams are kept current but not analysed, and files pause so that no content is lost.
- Files are read once, at their native frame rate, so a one-minute clip occupies one minute regardless of the sampling interval. At end of file the engine notifies clients (`ended`) and removes the source.
- A stream that fails is reconnected with exponential backoff, from 1 s doubling to a ceiling of 60 s, until it is removed. Clients are notified on failure (`lost`) and on recovery (`opened`). Adding an unreachable stream succeeds and reports the status `reconnecting`.
- Source lists are held in memory and cleared by a restart. The SDK re-registers the sources it added when it reconnects. The dashboard does not.

## SDK Reference

```python
stream = visionext.connect("ws://localhost:8765")
```

`connect()` waits for the engine to become available, reconnects after a restart or network interruption, and re-registers the sources added through it. Its arguments are listed under [`connect()` Options](#connect-options). Failures raise the exceptions under [Errors](#errors).

```python
stream.add_source(input, source_id=None, every_n_frames=1)   # returns a Source
stream.remove_source(source_id)                               # returns the removed Source
stream.list_sources()                                         # list[Source]
stream.sources                                                # dict[source_id, Source], kept current
stream.close()                                                # stop iterating, from any thread or callback

for record in stream: ...                                     # every sampled frame
for record in stream.detections(): ...                        # only frames with detections
for event in stream.events(): ...                             # detections, each with event.record
```

Source notifications (`added`, `removed`, `opened`, `lost`, `ended`) are delivered to an optional callback:

```python
def on_source(event, source):
    print(event, source.source_id, source.status)
    if event == "ended":
        stream.close()

stream = visionext.connect("ws://localhost:8765", on_source=on_source)
```

Persisting the frames that contain detections:

```python
for record in stream.detections():
    record.frame.save(f"hits/{record.source_id}/{record.frame_index:012d}.jpg")
```

The asyncio variant offers the same interface with awaitable methods:

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

| Exception | Raised when |
| --- | --- |
| `RequestError` | The engine rejected the request: an unreadable file (see [Troubleshooting](#troubleshooting)), a `source_id` already bound to a different input, or an unknown `source_id`. `str(exc)` gives the reason. |
| `RequestTimeout` | No reply arrived within `request_timeout` (default 30 s). Adding an unreachable stream can take up to 10 s. |
| `StreamClosed` | The connection closed while a request was pending. |
| `ConnectionError` | No engine was reachable within `connect_timeout` (default: wait indefinitely). |

Stream failures and file ends are not exceptions. They arrive as notifications through `on_source`.

### `connect()` Options

| Argument | Default | Meaning |
| --- | --- | --- |
| `reconnect` | `True` | Reconnect after an engine restart or network interruption and re-register this stream's sources. |
| `retry_every` | `2.0` | Seconds between connection attempts while the engine is unreachable. |
| `connect_timeout` | `None` | Raise `ConnectionError` after this many seconds without a connection. |
| `request_timeout` | `30.0` | Seconds to wait for a reply to a source request. |
| `on_source` | `None` | `callback(event, source)`. |

### `Record`

| Field | Meaning |
| --- | --- |
| `source_id` | The source the frame came from. |
| `timestamp` | Processing time as a timezone-aware `datetime` in UTC. |
| `frame_index` | 1-based position of the frame within the source, counting every frame read. With `every_n_frames=25` the values are 25, 50, 75, … |
| `frame_width`, `frame_height`, `size` | Pixel dimensions. |
| `events` | `list[Event]`. `objects` and `faces` filter by kind. Empty when nothing was detected. |
| `frame` | The unannotated frame as JPEG: `frame.bytes`, `frame.save(path)`, `frame.to_pil()` (requires Pillow), `frame.to_numpy()` (requires numpy and OpenCV, BGR). |

### `Event`

| Field | Meaning |
| --- | --- |
| `name` | `object.detected`, `face.detected` or `face.recognized`. Predicates: `is_object`, `is_face`, `recognized`. |
| `label` | The class name (`"person"`, `"dog"`, `"horse"`) for objects, the gallery identity or `"unknown"` for faces. |
| `confidence` | 0 to 1. Detections below the detector threshold of 0.5 are not emitted. |
| `bbox` | `BBox(x1, y1, x2, y2)` in pixels, with `width`, `height`, `area`, `center`, `rounded()` and `contains(x, y)`. |
| `class_id` | Objects only: the numeric class id. `1` is person, `16` to `25` are the animals. `visionext.COCO_LABELS` maps ids to names. |
| `identity`, `similarity`, `landmarks` | Faces only: the gallery identity (see [Face Gallery](#face-gallery)), the cosine similarity to it, and five `Point(x, y)` landmarks (eyes, nose, mouth corners). |
| `record` | The `Record` this event belongs to. |

### `Source`

`source_id`, `input` (credentials removed), `kind` (`file` or `stream`), `every_n_frames`, `status` (`open`, `reconnecting`, `ended`, `removed`), `frame_index`.

## Dashboard

<p align="center"><img src="screenshot.png" alt="The VisioNext dashboard showing live detections" width="900"></p>

The dashboard is a client of the engines in the same sense as your application. Its menu switches between **Object Detection** and **Face Recognition**, and each engine page offers:

- **Sources**: add a stream URL or an in-container file path with a sampling interval, and remove sources. The list is the engine's own, so sources registered through the SDK appear here as well.
- **Files**: upload a video of up to 4 GB and add it to the engine in one step. Uploads persist in the `uploads` volume, which every engine mounts, and remain addressable from the SDK as `/uploads/<name>` until deleted from the same dialog.
- **Live**: the latest sampled frame from each source with its detections overlaid. Selecting a frame opens the underlying record, its events and the JSON your application receives.
- **Records**: the most recent records, newest first, with their detections and confidence values.

Because the dashboard is a client, both engines analyse frames whenever the page is open in a browser. A file source therefore plays while you watch it, whether or not an application is connected.

## Face Gallery

`visionext-faces` identifies people from reference photos supplied in a directory mounted read-only at `/gallery`. Each sub-folder names one identity, which is reported as `event.identity`, and holds that person's photos:

```
gallery/
  alice/   front.jpg  side.jpg
  bob/     bob.png
```

- Accepted formats are `.jpg`, `.jpeg`, `.png`, `.bmp` and `.webp`. Use clear, frontal faces, two to five per person. Frames captured by the camera itself are suitable.
- The gallery is read once at startup, so changes require `docker compose restart faces`. The log line `Gallery: N identities from M images` in `docker compose logs faces` confirms what was loaded.
- Without a gallery the engine performs detection only. Every face is `"unknown"` and `event.recognized` is false.
- A match requires a similarity of 0.45 or higher. A person consistently reported as unknown with a similarity just below that threshold needs more, or clearer, photos.

## Security

The engine endpoint is unauthenticated, and the dashboard has no login. Any host that can reach an engine port can register sources and receive frames. Any host that can reach the dashboard can do the same and also upload videos. The compose file above therefore binds the engines to `127.0.0.1` and publishes only the dashboard on all interfaces, which is appropriate for a private network. Beyond that, bind the dashboard to `127.0.0.1` as well and place an SSH tunnel or an authenticating reverse proxy in front of it. Neither the engines nor the dashboard should be reachable from the public internet.

Stream credentials embedded in a source URL are stripped from every record, notification and log line the engine produces.

Face photos are biometric data. Establish consent and a legal basis before deploying recognition.

## Troubleshooting

| Symptom | Cause and remedy |
| --- | --- |
| `CUDA is required but unavailable` in `docker compose logs` | The GPU is not visible to the container. Verify `gpus: all`, the NVIDIA Container Toolkit installation, and that `nvidia-smi` works on the host. |
| The dashboard reports **Reconnecting** for an engine | The engine is still loading, or its compose service is not named `objects` or `faces`. |
| `connect()` never returns | The engine is still loading (`docker compose logs` prints `Listening on` when ready), the port is not published, or the host port is wrong. |
| Connected but no records arrive | No sources are registered (`stream.list_sources()`), or the source is `reconnecting`. Check the URL and credentials. |
| `RequestError: no such file inside the container: …` | The path is a host path, or the volume is not mounted. Mount the directory as shown under [Sources](#sources) and use the in-container path. |
| `Client … is not keeping up: dropping its oldest records` in the logs | The client consumes records more slowly than the engine produces them. Consume faster or raise `every_n_frames`. |
| Every face is `unknown` despite a gallery | Check the logs for `Gallery: N identities`. Zero means the directory is not mounted at `/gallery` or contains no readable faces. |
