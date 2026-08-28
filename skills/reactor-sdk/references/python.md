# Reactor Python SDK

Async, decorator-based API for scripts, servers, and data pipelines. A ctypes wrapper around a native `libreactor_ffi` library (Rust) — zero runtime Python dependencies. Frames arrive as NumPy arrays (or raw bytes) for direct processing.

## Install

```bash
pip install "reactor-sdk>=1.0"
```

Import as `reactor_sdk`. Pin `>=1.0` explicitly: unsupported platforms silently fall back to an old `py3-none-any` wheel (≤0.8.0) with a **different, older API** if you don't set a floor. Ships platform wheels only (Linux x86_64/aarch64, macOS arm64/x86_64, Windows x86_64) — no sdist.

Frame handling needs `numpy` (a real dependency of your code, not of the SDK). Audio capture/playback helpers need the `audio` extra: `pip install "reactor-sdk[audio]"` (pulls in `sounddevice`).

## Minimal end-to-end example

```python
import asyncio
import os
from reactor_sdk import Reactor, ReactorStatus

async def main():
    reactor = Reactor("helios", os.environ["REACTOR_API_KEY"])

    @reactor.on_status(ReactorStatus.READY)
    async def ready(status):
        await reactor.send_command("set_prompt", {"prompt": "a neon city"})

    @reactor.on_error
    def on_error(err):
        print(f"[{err.code}] {err.message}")

    async with reactor:
        track = reactor.tracks.with_kind("video").with_direction("recvonly").one()

        @track.on_frame
        def on_frame(frame):
            # frame: np.ndarray, shape (H, W, 3), dtype uint8, RGB
            pass

        await asyncio.Event().wait()  # run until cancelled

asyncio.run(main())
```

The SDK exchanges the API key for a JWT internally on `connect()` — no separate token-minting call needed server-side. See [authentication.md](authentication.md) for key handling and the `jwt=` alternative.

## Core API

| Method | Signature |
|---|---|
| Construct | `Reactor(model_name, api_key=None, *, jwt=None, api_url="https://api.reactor.inc", local=False)` |
| Connect | `await reactor.connect(*, session_id=None, connection_id=None)` |
| Reconnect | `await reactor.reconnect()` — reconnects the **same** session after a transient failure, without ending it server-side |
| Disconnect | `await reactor.disconnect()` — takes **no arguments**; always ends the session server-side, not recoverable |
| Close | `reactor.close()` — sync, destroys the local handle only, no network call |
| Send command | `await reactor.send_command(command, data)` → `dict \| None` — requires `ReactorStatus.READY` |
| Request schema | `await reactor.request_schema()` → the model's command schema as an OpenAPI document |
| Get/create a track | `reactor.track(name)` → `Track` |
| All declared tracks | `reactor.tracks` → `TrackList` |
| Paused tracks | `reactor.paused_tracks` → `frozenset[str]` |
| Publish track | `await reactor.publish_track(name)` → `Track` — prefer `track.publish()` |
| Unpublish track | `reactor.unpublish_track(name)` — sync, fire-and-forget; prefer `track.unpublish()` |
| Upload file | `await reactor.upload_file(file, *, name=None, mime_type=None)` → `FileRef` — only once `READY` |
| Status | `reactor.status` (property) / `reactor.get_status()` (legacy alias) |
| Session id | `reactor.session_id` (property) / `reactor.get_session_id()` (legacy alias) |
| Context manager | `async with reactor:` — connects on enter, disconnects + closes on exit |

`file` in `upload_file` can be a `str` path, `os.PathLike`, `bytes`, or a binary file-like object.

**`reconnect()` replaces the old "`disconnect(recoverable=True)` then `reconnect()`" dance entirely.** There is no `recoverable` parameter anywhere in this SDK — `disconnect()` always ends the session. For a transient network blip, just call `reconnect()` directly; it resumes the *same* session. Recvonly tracks resume automatically, but **sendonly tracks do not** — call `track.publish()` again after a `reconnect()`. See [Reconnection](#reconnection) below.

### What this SDK does *not* have

If you're carrying assumptions from another SDK generation or from Reactor's JS SDK, these do not exist here — don't guess at them:

- No `get_last_error()`, `get_capabilities()`, `get_remote_tracks()`, or `get_session_info()`. Read `reactor.tracks` / `reactor.track(name)` / `reactor.session_id` instead; capabilities have no public getter (they only drive internal track bookkeeping).
- No `MessageScope` / `APPLICATION` / `RUNTIME` concept. There's just `on_message` (application messages) and a `"runtime_message"` event reachable only via `reactor.on("runtime_message", handler)`.
- No `model_tracks` constructor parameter. Tracks are discovered from the model's declared capabilities after connecting.
- No `Reactor.on_frame` — it was removed in 0.9.0. Frame handling lives on `Track` (see [Frame handling](#frame-handling) below); calling `reactor.on("frame", ...)` or `reactor.on("audio", ...)` raises `ValueError` on purpose.
- No public `pause_track()` / `resume_track()` on `Reactor` — only `Track.pause()` / `Track.resume()`.

## Decorators for events

```python
@reactor.on_status(ReactorStatus.READY)
async def handle_ready(status): ...

@reactor.on_status(ReactorStatus.DISCONNECTED)
def handle_disconnect(status): ...

# Multiple statuses
@reactor.on_status([ReactorStatus.CONNECTING, ReactorStatus.WAITING])
def handle_setup(status): ...

# No filter — fires on every status change
@reactor.on_status
def handle_any(status): ...

@reactor.on_message
def handle_message(message):
    # Application messages the model broadcasts (a dict) — distinct from
    # send_command's own per-call resolved reply.
    ...

@reactor.on_track
def handle_track(track):
    # Fires once per named track the session declares. Hands the resolved
    # Track object itself, not a bare name — not filtered by name.
    ...

@reactor.on_error
def handle_error(error):
    # ReactorError — see Error handling below
    ...

# Generic registration for anything without a dedicated decorator
# (e.g. the runtime_message event — there is no @on_internal_message):
reactor.on("runtime_message", lambda msg: print(msg))
```

Register decorators **before** calling `connect()` (or entering `async with`) so you don't miss early events.

## Frame handling

Frame handling lives on `Track`, not on `Reactor`. Get a track via `reactor.track(name)`, or filter `reactor.tracks`:

```python
video_in = reactor.tracks.with_kind("video").with_direction("recvonly").one()

@video_in.on_frame
def on_frame(frame, frame_id, timestamp_us):
    # frame: np.ndarray (H, W, 3) uint8 RGB for video,
    #        or (samples, channels) int16 for audio.
    # Declare only the positional params you need — the SDK inspects the
    # handler's arity and passes that many.
    pass

@video_in.on_raw_frame
def on_raw_frame(bgra, width, height, frame_id, timestamp_us, user_data):
    # No numpy conversion — raw bytes, every argument always passed.
    pass
```

Callbacks run **synchronously, inline, on the native media-delivery thread** — this is deliberate backpressure: a slow handler drops/delays frames rather than queuing unboundedly. `async def` handlers are also supported and get scheduled onto the client's event loop via `run_coroutine_threadsafe`, but a plain `def` handler still runs inline. Offload real work either way:

```python
import asyncio
import queue

frame_queue: queue.Queue = queue.Queue(maxsize=32)

@video_in.on_frame
def enqueue(frame):
    try:
        frame_queue.put_nowait(frame)
    except queue.Full:
        pass  # drop if consumer is falling behind

async def consumer():
    loop = asyncio.get_running_loop()
    while True:
        frame = await loop.run_in_executor(None, frame_queue.get)
        await process(frame)  # heavy work here is fine
```

Control events (`on_status`, `on_error`, `on_message`, `on_track`) are marshalled onto the asyncio loop regardless — only frame/audio delivery bypasses it, for backpressure reasons.

`track.off_frame(handler)` removes a registered handler.

## Publishing video/audio input

Publish via the `Track` object, not `Reactor` directly — `track.publish()` returns the track itself for chaining:

```python
from reactor_sdk import Reactor, ReactorStatus, time_micros

reactor = Reactor("helios", api_key)

@reactor.on_status(ReactorStatus.READY)
async def start_publishing(status):
    track = (await reactor.publish_track("webcam"))  # "webcam" matches model input
    # or: track = await reactor.track("webcam").publish()

    track.push_frame(
        rgb_array,                       # numpy (H, W, 3) RGB, or (H, W, 4) BGRA, or raw BGRA bytes (+ width=/height=)
        capture_time_us=time_micros(),   # engine clock — NOT time.time(); required to align multiple sources to one moment
    )
```

`push_frame()` is one method for both video and audio — dispatch is by the track's own `kind`. Audio takes interleaved int16 PCM as `bytes` or a numpy int16 array (`sample_rate=`, `num_channels=`); `user_data=`/`capture_time_us=` raise `TypeError` on an audio track (no wire slot for them). Calling `push_frame()` before `publish()` raises `InvalidStateError`.

Wrong-direction calls are loud, not silent: `push_frame()` on a recvonly track, or `on_frame()`/`on_raw_frame()` on a sendonly track, raise `ValueError`.

`await track.unpublish()` (or `reactor.unpublish_track(name)`, sync) to stop.

## File uploads

For commands that take files, upload first and pass the returned `FileRef` into `send_command`. Only a **top-level** `FileRef` in `data` is detected and extracted as a separate upload reference — nested `FileRef`s are not:

```python
# From a path (name and MIME type inferred from filename)
ref = await reactor.upload_file("photo.jpg")

# From bytes (supply name so the runtime can infer MIME)
ref = await reactor.upload_file(image_bytes, name="photo.jpg")

# From a file-like object
with open("photo.jpg", "rb") as f:
    ref = await reactor.upload_file(f)

# Override MIME type explicitly
ref = await reactor.upload_file(blob, name="photo.jpg", mime_type="image/jpeg")

await reactor.send_command("set_image", {"image": ref, "strength": 0.7})
```

`upload_file()` only works once the session is `READY` — it raises `InvalidStateError` before that.

## Recording and clips

```python
clip = await reactor.request_clip(duration_seconds=5.0)   # last N seconds
# or: clip = await reactor.request_recording()             # entire session so far

# Download a Clip you already have (uses this client's own JWT, waits
# indefinitely by default until the manifest is ready or the session
# leaves READY):
data = await reactor.download(clip, "clip.mp4", on_progress=lambda fetched, total: ...)

# Or do request + download in one call:
data = await reactor.download_clip(duration_seconds=5.0, path="clip.mp4")
data = await reactor.download_recording(path="recording.mp4")
```

`Clip` is a frozen dataclass: `session_id, kind, start_marker, end_marker, now_marker, predicted_ready_at_ms, playlist_url`.

There's also a module-level `reactor_sdk.download_clip(clip, path=None, *, jwt=None, on_progress=None, ready_timeout=60.0, while_live=None)` for downloading a `Clip` with no live `Reactor` instance around (bounded 60s default timeout, since there's no session status to poll). It takes a `Clip`, not a duration — don't confuse it with `Reactor.download_clip()`, which takes a duration and requests one for you.

## Multiple connections to one session

`connect(session_id=..., connection_id=...)` lets a second client adopt an already-running session — useful for admin/observer connections or session hand-off between processes. `connection_id` is a `uint32`; out-of-range values raise `ValueError`.

## Reconnection

For a transient network blip, reconnect the same session directly — no prior `disconnect()` needed or wanted:

```python
await reactor.reconnect()
```

Raises `InvalidStateError` if there's no session to reconnect to. Recvonly tracks resume automatically; **re-publish** any sendonly track yourself:

```python
await reactor.reconnect()
await webcam_track.publish()
```

`disconnect()` always ends the session server-side — there's no way to disconnect "softly" in this SDK. If you need the process to end cleanly, prefer `async with reactor:` or an explicit `disconnect()` + `close()`; `__del__` and an `atexit` hook provide best-effort cleanup either way.

## Error handling

Errors surface through `@on_error` and are the exact same exception type a failed `await` raises — one class hierarchy, not two shapes that can disagree:

```python
class ReactorError(Exception):
    code: str              # e.g. "UNAUTHORIZED", "RATE_LIMITED" — open-ended, don't assume exhaustive
    message: str
    recoverable: bool
    status: int | None            # HTTP status, when applicable
    operation: str | None         # which call failed, e.g. "connect"
    retry_after_ms: float | None
    timestamp_ms: float | None
```

There is **no `component` field** — which platform tier failed isn't something a caller can act on.

Named subclasses (each just a fixed `.code`), all exported from `reactor_sdk`:

`NetworkError` (`NETWORK_ERROR`) · `UnauthorizedError` (`UNAUTHORIZED`) · `NotFoundError` (`NOT_FOUND`) · `ConflictError` (`CONFLICT`) · `RateLimitedError` (`RATE_LIMITED`) · `BadRequestError` (`BAD_REQUEST`) · `ServerError` (`SERVER_ERROR`) · `VersionMismatchError` (`VERSION_MISMATCH`) · `DecodeError` (`DECODE_FAILED`) · `InvalidStateError` (`INVALID_STATE`) · `SessionTerminalError` (`SESSION_TERMINAL`) · `MessageTooLargeError` (`MESSAGE_TOO_LARGE`) · `TransportError` (`TRANSPORT_ERROR`) · `DisconnectedError` (`DISCONNECTED`) · `RequestTimeoutError` (`REQUEST_TIMEOUT`) · `AbortedError` (`ABORTED`)

An unrecognized platform code falls back to the base `ReactorError` with `.code` set to whatever the platform sent — never assume an unfamiliar code means something is broken.

```python
import asyncio
from reactor_sdk import ReactorError, UnauthorizedError

@reactor.on_error
async def handle_error(err: ReactorError):
    if isinstance(err, UnauthorizedError):
        raise SystemExit("bad API key / JWT")
    print(f"[{err.code}] {err.message}")
    if err.recoverable:
        await asyncio.sleep((err.retry_after_ms or 3000) / 1000)
        await reactor.reconnect()
```

Dispatch on `isinstance(err, SomeError)` or `err.recoverable` / `err.retry_after_ms` rather than string-matching `err.code` — same reasoning as the JS SDK.

**Separate exception, don't conflate with `ReactorError`:** `AuthError` (`RuntimeError`, not a `ReactorError` subclass) is raised when exchanging an `api_key` for a JWT fails (bad key, unreachable auth host, malformed response) — catch it separately around `connect()`.

Don't raise from inside `@on_error` — the handler runs on the event loop.

## Env vars

Convention: `REACTOR_API_KEY`. Pass it explicitly to the constructor (`Reactor("helios", os.environ["REACTOR_API_KEY"])`) — there is no implicit env-var fallback in the SDK itself. For minting JWTs on behalf of browser clients instead, see [authentication.md](authentication.md).

## Version note

`reactor-sdk` is published on PyPI (`pip install reactor-sdk`), currently at 1.1.1, with **no `CHANGELOG.md`** in the repo — there's nothing to link for a version history. This reference reflects 1.1.1's actual source as of the last time this skill was verified; re-check `reactor.__version__` / the installed source against this doc if something doesn't match.
