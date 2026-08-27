---
name: reactor-sdk
description: Build real-time video AI applications with Reactor SDKs. Use when code imports `@reactor-team/js-sdk` or `reactor-sdk`; when connecting a web/mobile frontend to a GPU-powered Reactor model (Helios and others); when streaming video via WebRTC with command-based control; when publishing webcam input for video-to-video transformations; when handling JWT authentication for Reactor; or when debugging Reactor connection lifecycle (disconnected → connecting → waiting → ready). SKIP when project uses unrelated WebRTC (e.g., plain video calling), or video pipelines that don't touch Reactor.
---

# Reactor SDK

Reactor streams real-time video from GPU-hosted AI models to web and mobile frontends over WebRTC. Clients authenticate with JWTs, connect to a named model, send commands to control generation, and receive video tracks (JS/React) or NumPy frames (Python).

**SDKs:**
- JavaScript / TypeScript: `@reactor-team/js-sdk` (npm / pnpm) — this skill documents **3.0.0**, a wasm-bindgen rewrite over `reactor-core` that is a major, non-backward-compatible bump from the 2.x line (built directly on `RTCPeerConnection`). **3.0.0 is not published to npm yet** — `npm install` currently gets a 2.x release. Check the installed `package.json` version before trusting method names/return types from this skill; if it's `<3.0.0`, only the connection-lifecycle basics still apply.
- Python: `reactor-sdk` (pip) — currently **1.1.1**, published on PyPI, no gating caveat. This is a full rewrite from any older generation: a `ctypes` wrapper over a native library, zero runtime Python dependencies.

**API key format:** `rk_...` — never commit or expose in client bundles.

## Pick the right reference

Load only the reference(s) matching the task. Each is self-contained.

| Task | Read |
|---|---|
| Vanilla JS / TS app, non-React framework | [references/javascript.md](references/javascript.md) |
| React app (hooks, `ReactorProvider`, `ReactorView`, `WebcamStream`, `ClipPlayer`) | [references/react.md](references/react.md) |
| Python script, server, data pipeline, frame processing | [references/python.md](references/python.md) |
| JWT fetching, dev vs. production auth, env vars | [references/authentication.md](references/authentication.md) |
| Model-specific command schemas, model roster, scaffolding | [references/models.md](references/models.md) |

For tasks that span SDKs (e.g., a React frontend talking to a Python backend), read both references — and note that JS and Python have diverged more than you'd expect (see Critical gotchas below); don't assume a pattern that works in one applies to the other.

## Connection lifecycle

Every SDK exposes the same four-state lifecycle:

```
disconnected → connecting → waiting → ready
```

| State | Meaning | Can send commands? |
|---|---|---|
| `disconnected` | No active connection | No |
| `connecting` | Establishing coordinator connection | No |
| `waiting` | Queued for GPU assignment | No |
| `ready` | GPU assigned, session live | **Yes** |

**Always wait for `ready` before sending commands or publishing tracks.** Commands sent earlier are rejected. Register event listeners / decorators *before* calling `connect()` so you don't miss the `ready` transition.

## Core workflow (SDK-agnostic)

1. Create an API key (`rk_...`) in the [Reactor Dashboard](https://reactor.inc/dashboard).
2. **JS**: mint a JWT server-side by POSTing to `https://api.reactor.inc/tokens` with the `Reactor-API-Key` header; ship only the JWT to the browser (pass it via the constructor's `jwt` option or `connect(jwt)`). **Python**: pass the API key directly to the `Reactor` constructor (`Reactor(model_name, api_key)`) — the SDK exchanges it for a JWT internally.
3. Create a `Reactor` instance with a model name (`modelName` in JS, first positional arg in Python). Tracks are advertised by the server — the client does not declare `send` / `receive` arrays, and JS's constructor doesn't take a track list either beyond the optional escape-hatch `modelTracks`.
4. Register event listeners (JS: `reactor.on(...)`) / decorators (Python: `@reactor.on_status`, etc.) — before `connect()`.
5. `connect()`.
6. On `ready`, send commands and/or publish tracks (names match the model's input attributes). **Both SDKs now await the actual reply** from `sendCommand`/`send_command` — see the gotcha below, this is the single biggest behavior change from older docs/memory of this SDK.
7. On teardown: JS `disconnect()`; Python `disconnect()` (no args, always ends the session) or `close()`/`async with` for cleanup. Stop any local media tracks yourself either way.

SDK-specific API calls for each step are in the per-SDK references.

## Critical gotchas

These bite every integration. Read the ones marked JS/Python carefully — the two SDKs have diverged significantly from each other and from any prior generation of this skill.

- **`sendCommand`/`send_command` now await the actual reply, in both SDKs.** This used to be fire-and-forget (`Promise<void>` in JS that resolved almost immediately; no return value worth reading in older Python). Now: JS resolves `Promise<ReactorMessage | undefined>`, Python resolves `dict | None`, both bounded by a timeout. Code written against the old fire-and-forget assumption still works (nobody's forced to read the reply), but code that assumed the promise/coroutine resolved *before* the model actually processed anything is now wrong — it resolves after the round trip. **JS-specific exception:** `sendCommand()` itself never rejects even on failure — it routes failures through `getLastError()`/the `error` event and resolves `undefined`. This is the one JS call that doesn't throw; every other throwing call (`connect`, `publishTrack`, `uploadFile`, `requestClip`, ...) behaves normally. Python's `send_command` raises like anything else.
- **`ReactorError` is a typed class hierarchy in both SDKs now, not a flat `{code, message, component, ...}` object/dataclass.** `component: "api" | "gpu"` is **gone entirely** in both — dispatch on `instanceof`/`isinstance` against named subclasses (`UnauthorizedError`, `RateLimitedError`, `ConflictError`, ...) or on `.recoverable`/`.retryAfter` (JS, plus a `retry_after_ms` canonical field) / `.retry_after_ms` (Python) instead of matching `code` strings — the code vocabulary is intentionally open-ended. See the per-SDK references' Error handling sections for the full class list; JS and Python's lists are the same 16-17 names.
- **Python's connection-recovery story changed completely.** There is no `recoverable=` parameter anywhere in the Python SDK. `disconnect()` takes **no arguments** and always ends the session non-recoverably. To recover from a transient blip, call `reconnect()` directly — it resumes the *same* session without a prior disconnect. Recvonly tracks resume automatically; **re-publish sendonly tracks yourself** after a Python `reconnect()`. JS kept the old shape: `disconnect(true)` keeps the session alive, then `reconnect()` resumes it. **Don't port the JS pattern to Python or vice versa** — they now work differently, not just differently-named.
- **Python moved frame handling and pause/resume off `Reactor` entirely, onto `Track`.** `Reactor.on_frame` was removed (0.9.0) — get a track via `reactor.track(name)` and use `track.on_frame`/`track.on_raw_frame`/`track.push_frame`. Same for `pause_track`/`resume_track` → `track.pause()`/`track.resume()`. Calling `reactor.on("frame", ...)` raises `ValueError` on purpose. JS didn't make this move — `pauseTrack`/`resumeTrack` stay on `Reactor` (and on the React store).
- **Python has no `MessageScope`.** JS still has `"application"`/`"runtime"` as a `sendCommand` argument. Python splits this by *event* instead — `on_message` for application messages, and an unadorned `reactor.on("runtime_message", handler)` for the runtime channel (no dedicated decorator).
- **Python's `Reactor` has a much smaller state-accessor surface than you might expect from JS.** No `get_last_error()`, `get_capabilities()`, `get_remote_tracks()`, `get_session_info()`, or a `model_tracks` constructor param — none of these exist. Use `reactor.tracks` / `reactor.track(name)` / `reactor.session_id` / `@on_error` instead. Don't assume JS's fuller accessor list has a Python equivalent — check [python.md](python.md) before guessing.
- **Recording/clips are new in both SDKs and have no prior-generation equivalent.** JS: `reactor.requestClip()`/`requestRecording()`/`downloadClipAsFile()`, plus React's `ClipPlayer`/`ClipDownloadButton`/`useClipDownload`. Python: `reactor.request_clip()`/`request_recording()`/`download()`/`download_clip()`/`download_recording()`, plus a module-level `reactor_sdk.download_clip()` with a different signature (takes a `Clip`, not a duration — don't confuse the two `download_clip`s). If a prior integration or memory of this skill has no mention of clips/recording, that's not because it was omitted — the feature didn't exist yet.
- **Never expose a raw API key in browser code.** Mint JWTs server-side via `POST https://api.reactor.inc/tokens` and ship only the JWT. See [references/authentication.md](references/authentication.md).
- **Track names are contracts.** The string passed to `publishTrack()` / `publish_track()` (or Python's `track.publish()`) must match the server-side model's input attribute name exactly. Mismatches fail silently — no error, no frames.
- **JWTs expire after 6 hours.** Long-lived sessions need refresh + reconnect logic.
- **Media cleanup is mandatory.** JS: forgetting `unpublishTrack()` + `stream.getTracks().forEach(t => t.stop())` leaks cameras/memory across route changes or unmounts (`unpublishTrack` itself never rejects, so it's safe in a `finally`/cleanup path). Python: `track.unpublish()` and stop your own capture source.
- **Commands are model-specific.** Command names and argument shapes differ per model. Check [references/models.md](references/models.md) or fetch the model's own schema (`schemaReceived`/`getSchema()` in JS, `await reactor.request_schema()` in Python) before guessing.
- **React store renames two methods.** `useReactor` exposes `publish` / `unpublish` (not `publishTrack` / `unpublishTrack` — those names exist on the underlying `Reactor` class, hook consumers use the shorter names). The React store also exposes `pauseTrack`/`resumeTrack`/`requestClip`/`requestRecording`/`downloadClipAsFile` directly as actions.
- **Cross-SDK naming still drifts, even beyond the error hierarchy.** Status values are lowercase strings in JS (`"ready"`) and a `str` enum in Python (`ReactorStatus.READY`, which does compare equal to `"ready"`). JS keeps camelCase compat aliases (`retryAfter`, `timestamp`) alongside the canonical snake_case fields (`retry_after_ms`, `timestamp_ms`); Python only has the snake_case forms. When porting code between SDKs, don't assume a straight case-convention swap is enough — check the actual field/method list in the target SDK's reference.

## Verification checklist

Before shipping Reactor integration code:

- [ ] API key lives in env vars or a secret manager, never in source
- [ ] Browser code fetches the JWT from your server, not from a client-exposed key
- [ ] Listeners/decorators are registered *before* `connect()`
- [ ] All command sends are gated on `status === "ready"` / `ReactorStatus.READY`
- [ ] The resolved reply from `sendCommand`/`send_command` is either used or deliberately ignored — not assumed to be `void`/synchronous-fire-and-forget
- [ ] Teardown path calls `disconnect()` (Python: no args) and stops every local media track
- [ ] Error handling dispatches on the `ReactorError` subclass / `.recoverable` / `.retry_after_ms`, not on specific `code` strings
- [ ] Track names passed to `publishTrack()` / `publish_track()` match the model's input attribute
- [ ] Python frame handlers (now on `Track`, not `Reactor`) are non-blocking, or async where that fits better
- [ ] Python reconnection uses `reconnect()` directly (no `disconnect(recoverable=True)` — that shape doesn't exist here); JS reconnection uses `disconnect(true)` + `reconnect()`
- [ ] Full lifecycle exercised once end-to-end: connect → ready → command (and its reply, if used) → disconnect

## Official docs

- Navigation index: https://docs.reactor.inc/llms.txt
- Auth guide: https://docs.reactor.inc/authentication
- JS SDK guide: https://docs.reactor.inc/javascript-guide
- Python SDK guide: https://docs.reactor.inc/python-guide
- Model catalog: https://docs.reactor.inc/model-api-reference/overview

When in doubt, fetch the current model or SDK page — API surfaces drift and this skill may be stale. Verify method names and argument shapes against the live docs, or against the actual installed package version, before writing non-trivial code.
