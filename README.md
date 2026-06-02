# inference-streaming-benchmark

Benchmark implementation for image transmission and inference response latency using one AI inference server across multiple transport protocols: HTTP multipart (FastAPI), ZeroMQ (ZMQ), ImageZMQ, and gRPC.

![Central operator panel — live aggregate, latency breakdown, and transport head-to-head](docs/assets/ui-operator-overview.png)

![Sweep configuration and connected-clients grid](docs/assets/ui-operator-sweep-clients.png)

The central operator UI can sweep transport and batching configurations, aggregate benchmark results, and compare per-client performance across active backends.

Results for 1920×1080 images on a MacBook Pro M2 Pro (YOLOv8n, MPS inference).

The columns are ordered so the math reads left-to-right:

```
enc + dec + comms  =  transport     (network + codec overhead)
                +  infer + post  =  total          (server-side processing)
```

`post` here is the server-side JSON serialization of detection results for the response, **not** YOLO post-processing (which is already counted inside `infer`).

| Backend            | Frames | Duration (s) | FPS  | enc (ms) | dec (ms) | comms (ms) | **transport (ms)** | infer (ms) | post (ms) | **total (ms)** |
| ------------------ | ------ | ------------ | ---- | -------- | -------- | ---------- | ------------------ | ---------- | --------- | -------------- |
| imagezmq           | 149    | 4.5          | 33.0 | 0.0      | 0.0      | 3.1        | 3.1                | 26.0       | 1.3       | 30.6           |
| zmq_raw            | 81     | 2.5          | 31.8 | 0.2      | 0.0      | 3.6        | 4.1                | 26.1       | 1.2       | 31.4           |
| grpc               | 93     | 6.1          | 15.4 | 0.2      | 0.2      | 6.6        | 7.2                | 22.4       | 0.9       | 30.4           |
| websocket_raw      | 271    | 9.3          | 29.1 | 0.3      | 0.0      | 7.2        | 7.6                | 25.0       | 1.3       | 34.2           |
| http_multipart_raw | 135    | 4.6          | 29.3 | 0.2      | 0.0      | 7.6        | 7.9                | 24.0       | 1.2       | 33.2           |
| http_multipart     | 115    | 5.1          | 22.4 | 5.6      | 8.6      | 3.8        | 18.4               | 23.9       | 1.1       | 43.4           |
| zmq                | 117    | 5.2          | 22.5 | 6.6      | 11.0     | 0.9        | 18.6               | 24.3       | 1.4       | 44.4           |
| websocket          | 78     | 3.5          | 22.6 | 6.6      | 11.0     | 1.0        | 18.8               | 23.9       | 1.3       | 44.0           |


## Setup

Requirements: Python 3.10+. [uv](https://docs.astral.sh/uv/) is recommended — it reads `pyproject.toml` and `uv.lock` for a reproducible install.

```bash
git clone https://github.com/TimoIllusion/inference-streaming-benchmark.git
cd inference-streaming-benchmark

# Recommended (uv, locked via uv.lock):
uv sync --all-extras

# Or classic pip:
python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev,test]"
```

> Note: Install torch with the appropriate hardware acceleration backend, such as CUDA or MPS, to improve inference speed.

## Run

One AI server process hosts every transport. A connected client picks which protocol is active; the server hot-swaps listeners on demand. Exactly one transport is active at any time.

```bash
python server.py        # control plane + central UI on :9000, http_multipart active by default
python client.py        # http://127.0.0.1:8501 (webcam UI, backend dropdown, mock-camera toggle)
```

Open `http://127.0.0.1:9000/` for the **central operator panel** — registers connected clients, switches transport for everyone with one click, toggles per-client mock camera and inference.

**Multiple clients on one machine**: pass `--port` (or `--port 0` for auto-pick). The default client name auto-suffixes the port (`<hostname>-<port>`) so they don't collide in the registry:

```bash
python client.py                          # 8501, name = <hostname>-8501
python client.py --port 8502              # 8502, name = <hostname>-8502
MOCK_CAMERA=1 python client.py --port 0   # any free port, mock frames
```

Options:
- `python server.py --default zmq` — start with `zmq` active
- `python server.py --default none` — idle until a client picks a transport

| Transport      | Port  | Description                       |
| -------------- | ----- | --------------------------------- |
| http_multipart | 8008  | FastAPI + multipart/form-data     |
| zmq            | 5555  | ZeroMQ REQ/REP + JPEG             |
| imagezmq       | 5556  | ImageZMQ + raw ndarray            |
| grpc           | 50051 | gRPC unary + raw ndarray bytes    |

> Note: Image transfer speed can improve significantly if frames are resized before sending. This usually does not affect model quality when the model already expects smaller inputs, such as 224x224.

## Multi-device deployment

The server (workstation/GPU box) and clients (RPi5s, laptops, anything with Python) can live on different machines on the same LAN.

**Server (workstation):**
```bash
python server.py
# central UI: http://<server-ip>:9000/
```

**Each client device (one per machine):**
```bash
INFSB_CONTROL_HOST=<server-ip> INFSB_CLIENT_NAME=rpi-edge-1 python client.py
# per-device UI: http://<client-ip>:8501/
```

Clients auto-register with the server on startup and send a heartbeat every second; the central UI shows them in a live table with FPS, latency, and per-client toggles.

**No webcam attached?** Set `MOCK_CAMERA=1` to start with synthetic frames (a static DALL-E image, looped at 30fps), or toggle the **Mock camera** switch in the per-device UI / central UI at runtime — no restart needed.

### Network exposure

This is a **benchmark tool for trusted LANs**. By default, the control plane (`:9000`) and per-device UI (`:8501`) bind `0.0.0.0` and run **without authentication** — the multi-device flow is the whole point of the project, so locking it down by default would only get in the way. The control plane holds no user data, no credentials, and no production traffic; the worst-case impact of unauthenticated access from the LAN is "someone else can disrupt your benchmark run."

Two opt-in escape hatches are available for hostile networks or CI:

- `INFSB_BIND=127.0.0.1` — restrict the control plane and per-device UI listeners to loopback. Set on the host you want to lock down. Transport sockets (HTTP multipart, ZMQ, gRPC, WebSocket, ImageZMQ) keep binding `0.0.0.0` since the multi-device benchmark depends on it.
- `INFSB_TOKEN=<random>` — require `Authorization: Bearer <token>` on every non-loopback request to the control plane and per-device UI. Loopback always passes through, so local `curl`/dev workflows stay simple. Outbound calls (server → client, client → server, sweep CLI) read the same env var and send the header automatically — set the same value on **every** machine.

Generate a token with:

```bash
python -c 'import secrets; print(secrets.token_urlsafe(32))'
```

> The token is for accident prevention, not for defending against motivated adversaries: there is no TLS, no per-client auth, no rate limiting. If you need real security, run the cluster behind a VPN or SSH tunnel.

**Per-transport multi-client behavior** (Option 1: each transport keeps its natural semantics):
- HTTP multipart, gRPC, WebSocket — fan in concurrently at the network layer; serialize at inference (one shared YOLO instance).
- ZMQ REQ/REP, ImageZMQ — strictly 1:1 by design. With N clients connected, the server processes one client at a time and the others queue. The central UI surfaces this clearly: one client at full FPS, others starved.

### Dynamic batching

The server has a built-in batcher in front of the inference engine. When **enabled**, concurrent `infer()` calls are coalesced into a single `model([img1, img2, ...])` call so the GPU gets fed `N` frames per invocation instead of `1`. Aggregate throughput rises with the number of clients sending at once.

Toggle from the **central UI** (top control card): `Dynamic batching` switch + `max size` + `max wait (ms)` + Apply. Or from the API: `POST /batching {enabled, max_batch_size, max_wait_ms}`. Disabled by default — pass-through to `engine.infer` with zero overhead, so existing single-stream numbers stay valid.

Two new columns appear in every stats table:
- **`wait (ms)`** — total pre-inference wait before the model call started. Zero when batching is off. In logs this is split into `backlog_wait` (queued behind earlier inference work) and `batch_fill_wait` (waiting for more frames in the current batch window), so total wait can exceed `max wait` when the worker is already busy.
- **`batch`** — median batch size across the frames recorded for that backend. `1` when batching is off or only one client is in flight.

The `transport (ms)` column is `total − infer − post − wait`, so it stays "network + codec only" even with batching on.

**Caveat for ZMQ/ImageZMQ:** these transports submit one frame at a time (REQ/REP semantics), so the batcher never coalesces — it just adds the `max_wait_ms` timeout penalty. The `batch` column staying at `1` and `wait (ms)` matching `max_wait_ms` makes this visible. Leave batching off for those transports unless you're explicitly studying the trade-off.

### Inference concurrency modes

The server can run YOLO inference in three modes, configurable from the central UI or `POST /inference {mode, instances}`:

- **`single`** — one shared YOLO instance with serialized model calls. This is the default and most reproducible mode.
- **`unsafe-multi`** — one shared YOLO instance with concurrent model calls. This restores the old faster-but-undocumented behavior.
- **`multi-instance`** — multiple YOLO instances, with at most one in-flight call per instance. This isolates model calls but uses more memory and startup time.

Inference mode and instance count are included in benchmark grouping and sweep results.

### Multi-run benchmark sweeps

With the server and one or more clients already running, sweep transports and batching configs from the control plane:

```bash
uv run infsb-multi-run \
  --transports http_multipart_raw,grpc,websocket_raw \
  --batch off,on \
  --batch-sizes 1,4,8 \
  --batch-waits-ms 0,5,10 \
  --duration-s 10 \
  --warmup-s 2 \
  --output benchmark-runs.json
```

Each run applies the batching config, switches all active clients to the selected transport with inference on, warms up, clears stats, measures for the configured duration, then stores the `/clients` snapshot in the output JSON.

| Env var               | Default              | Purpose                                      |
| --------------------- | -------------------- | -------------------------------------------- |
| `INFSB_CONTROL_HOST`  | `localhost`          | Server hostname/IP the client should reach   |
| `INFSB_CONTROL_PORT`  | `9000`               | Server control plane + central UI port       |
| `INFSB_UI_PORT`       | `8501` (auto-fallback) | Per-device client UI port. CLI `--port` overrides. |
| `INFSB_CLIENT_NAME`   | `<hostname>-<port>`    | Friendly name shown in the central UI. CLI `--name` overrides. |
| `MOCK_CAMERA`         | unset                | `1` → start with synthetic frames           |
| `INFSB_BATCH_ENABLED` | `0`                  | `1` → start the server with dynamic batching on |
| `INFSB_BATCH_SIZE`    | `8`                  | Max frames per batched model call            |
| `INFSB_BATCH_WAIT_MS` | `10`                 | Max time the batcher waits to fill a batch  |
| `INFSB_INFER_MODE`    | `single`             | Inference mode: `single`, `unsafe-multi`, or `multi-instance` |
| `INFSB_INFER_INSTANCES` | `2`                | YOLO instance count used by `multi-instance` mode |
| `INFSB_TRANSPORT_PORT_<NAME>` | transport default | Override a transport listener port, e.g. `INFSB_TRANSPORT_PORT_HTTP_MULTIPART=18008` |
| `INFSB_BIND`          | `0.0.0.0`            | Bind host for the control plane + per-device UI. Set to `127.0.0.1` to restrict to loopback. |
| `INFSB_TOKEN`         | unset                | Shared secret. When set, non-loopback requests must carry `Authorization: Bearer <token>`. |

### Adding a new transport

1. Create `inference_streaming_benchmark/transports/<name>/transport.py` with a `class XxxTransport(Transport)` implementing `start` / `stop` / `connect` / `send` / `disconnect`.
2. In `inference_streaming_benchmark/transports/<name>/__init__.py`, add `register(XxxTransport)`.
3. Add `from . import <name>` to `inference_streaming_benchmark/transports/__init__.py`.

That's it — the client backend dropdown and server control plane pick it up automatically.

## Tests

```bash
pip install -e ".[test]"
```

```bash
pytest tests
```
## Docker

```bash
docker build -f ./docker/Dockerfile -t inference-streaming-benchmark:latest .
```

```bash
# Runs `python server.py` (http_multipart active by default).
# Expose :9000 (control plane) plus whichever transport port you want to reach.
docker run -it --name aiserver1 --rm --shm-size=8g --gpus=all \
  -p 9000:9000 -p 8008:8008 \
  inference-streaming-benchmark:latest

# To start with a different default transport, pass through the flag and expose its port:
docker run -it --rm --shm-size=8g --gpus=all \
  -p 9000:9000 -p 5556:5556 \
  inference-streaming-benchmark:latest python server.py --default imagezmq
```

## Versioning

Versions are derived from git tags via [setuptools-scm](https://github.com/pypa/setuptools_scm). Every merge to `main` triggers `.github/workflows/auto-tag.yml`, which creates the next tag (e.g. `v0.1.5` → `v0.1.6`).

The default bump is **patch**. Override by including one of these keywords in a commit message of the merged range:

| Keyword  | Effect                        |
| -------- | ----------------------------- |
| `#minor` | minor bump (`v0.1.5` → `v0.2.0`) |
| `#major` | major bump (`v0.1.5` → `v1.0.0`) |
| `#none`  | skip tagging for this merge   |

For squash-merged PRs, putting the keyword in the PR title or description works, since GitHub includes both in the resulting commit message. For merge commits, put it in a branch commit or edit the merge commit message.

## Common Issues

- Camera not working and throwing errors → close all other browser tabs pointing at `http://127.0.0.1:8501/` (the per-device client UI) and reload the remaining one. Only one tab can hold the webcam at a time.

## Work Tracking

Open work is tracked with beads. Run `bd ready` to see available issues.

## AI Assistance

Development of this project was supported by OpenAI's and Claude's AI coding agents, e.g. Opus 4.8 with Claude Code and Claude Design or GPT-5.5 with OpenAI Codex. The goal of this repository was also to understand and explore the capabilities of these state of the art coding tools.
