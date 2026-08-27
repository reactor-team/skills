# Reactor JavaScript / TypeScript SDK

Imperative API for vanilla JS, TS, and non-React frameworks. React wrappers live in [react.md](react.md).

> **Version note:** this reference documents `@reactor-team/js-sdk` **3.0.0** (built on `reactor-wasm`, wasm-bindgen over `reactor-core`) — a major, non-backward-compatible rewrite of the 2.x line (built directly on `RTCPeerConnection`). **3.0.0 is not published to npm yet.** `npm install @reactor-team/js-sdk` currently installs a 2.x release. Check the installed `package.json` version before assuming this reference applies — if it's `<3.0.0`, the method names below are mostly right but `sendCommand()`'s return value and the entire error-handling section are wrong for that install. Once 3.0.0 ships, this note goes away.

## Install

```bash
npm install @reactor-team/js-sdk
# or
pnpm add @reactor-team/js-sdk
```

## Minimal end-to-end example

```ts
import { Reactor, UnauthorizedError } from "@reactor-team/js-sdk";

const reactor = new Reactor({
  modelName: "helios",
  jwt: () => fetchToken(),   // called once per connect(); see authentication.md
});

reactor.on("statusChanged", async (status) => {
  if (status === "ready") {
    // Command args are model-specific; verify against docs.reactor.inc/model-api-reference/<model>/schema
    await reactor.sendCommand("set_prompt", { prompt: "a neon city" });
  }
});

reactor.on("trackReceived", (name, track, stream) => {
  if (name !== "main_video") return;
  const el = document.querySelector<HTMLVideoElement>("#output")!;
  el.srcObject = stream;
  el.play();
});

reactor.on("error", (error) => {
  console.error(error.code, error.message);
});

try {
  await reactor.connect();
} catch (error) {
  if (error instanceof UnauthorizedError) {
    // bad/expired JWT
  } else if (error instanceof Error) {
    throw error;
  }
}

// Teardown — ends the session AND frees the wasm client
await reactor.disconnect();
```

Gate command sends on the `statusChanged` event rather than polling `getStatus()` — listeners must be registered before `connect()` or early transitions are missed.

Tracks are advertised by the server via the model's declared capabilities; the client does not declare `receive` or `send` arrays.

Construction never touches WebAssembly — the wasm module is fetched/instantiated lazily on the first `connect()`/`reconnect()` and cached after that.

## Core API

| Method | Signature |
|---|---|
| Construct | `new Reactor(options: ReactorOptions)` — only `modelName` is required |
| Connect | `reactor.connect(jwt?: JwtSource, options?: ConnectOptions): Promise<void>` |
| Reconnect | `reactor.reconnect(options?: ConnectOptions): Promise<void>` — resumes after `disconnect(true)`; only `options.maxAttempts` has effect |
| Disconnect | `reactor.disconnect(recoverable = false): Promise<void>` — ends the session server-side either way; `true` keeps the local wasm client alive for `reconnect()` |
| Dispose | `reactor[Symbol.dispose]()` — supports `using reactor = new Reactor(...)`; frees the wasm graph and drops every handler permanently |
| Send command | `reactor.sendCommand(command, data?, scope?): Promise<ReactorMessage \| undefined>` — requires `ready`; **never rejects** (see below) |
| Request schema | `reactor.requestSchema(): Promise<ModelSchema \| undefined>` — most apps don't need this; auto-fetched on `"ready"` (read via `getSchema()` or the `schemaReceived` event) |
| Publish track | `reactor.publishTrack(name: string, track: MediaStreamTrack): Promise<void>` |
| Unpublish track | `reactor.unpublishTrack(name: string): Promise<void>` — does **not** reject; failures report via the `error` event (safe to call in a `finally`) |
| Pause / resume track | `reactor.pauseTrack(name): Promise<void>` / `reactor.resumeTrack(name): Promise<void>` |
| Upload file | `reactor.uploadFile(file: File \| Blob, options?: { name?: string }): Promise<FileRef>` |
| Request clip / recording | `reactor.requestClip(durationSeconds: number): Promise<Clip>` / `reactor.requestRecording(): Promise<Clip>` |
| Download a clip | `reactor.downloadClipAsFile(clip, filename?, options?): Promise<Blob>` — does **not** inherit the instance's own JWT, pass one via `options.jwt` |
| Register listener | `reactor.on(event, handler)` / `reactor.off(event, handler)` / `reactor.once(event, handler)` |

`ConnectOptions` is `{ sessionId?, connectionId?, autoResumeTracks?, maxAttempts? }`. `MessageScope` is `"application"` (default; model commands) or `"runtime"` (platform-level — only `requestSchema`/`requestCapabilities` route anywhere; anything else warns and falls back to an application-scope send).

`publishTrack`/`unpublishTrack`/`uploadFile` are queued behind any in-flight `connect()`/`disconnect()`/`reconnect()` rather than racing them.

### State accessors

| Method | Returns |
|---|---|
| `reactor.getStatus()` | `ReactorStatus` — `"disconnected" \| "connecting" \| "waiting" \| "ready"` |
| `reactor.getSessionId()` | `string \| undefined` |
| `reactor.getLastError()` | `ReactorError \| undefined` |
| `reactor.getCapabilities()` | `Capabilities \| undefined` — cached, pushed automatically once negotiated |
| `reactor.getSchema()` | `ModelSchema \| undefined` — cached, auto-fetched on `"ready"` |
| `reactor.getSessionInfo()` | `SessionResponse \| undefined` — **raw wire shape** (snake_case, e.g. `session_id`), live/uncached. Prefer `getCapabilities()` over reading `.capabilities` off this. |
| `reactor.getStats()` | `ConnectionStats \| undefined` — refreshed every 2s while `ready` |
| `reactor.getConnectionTimings()` | `{ sessionCreationMs, transportConnectingMs, totalMs } \| undefined` |
| `reactor.getJwtResolver()` | The `JwtSource` this instance was constructed/last-connected with |
| `reactor.tracks()` | `TrackCapability[]` — every track the model declared, whether or not media has arrived |
| `reactor.trackMapping()` | `TrackMappingEntry[]` — `tracks()` plus the negotiated `mid` |
| `reactor.pausedTracks()` | `string[]` |
| `reactor.getTrackByName(name)` / `getStreamByName(name)` | `MediaStreamTrack \| MediaStream \| undefined` |
| `reactor.getTrackByMid(mid)` / `getStreamByMid(mid)` | Escape hatches keyed by negotiated `mid` |
| `reactor.getPeerConnection()` | `RTCPeerConnection \| undefined` — raw escape hatch |

## Events

```ts
reactor.on("statusChanged", (status: ReactorStatus) => {});
reactor.on("sessionIdChanged", (id: string | undefined) => {});
reactor.on("trackReceived", (name, track, stream, mid) => {});  // track/stream always defined
reactor.on("message", (message: ReactorMessage) => {});         // application-scope, from the model
reactor.on("runtimeMessage", (message: ReactorMessage) => {});  // platform-scope (moderation, clip/recording lifecycle, ...)
reactor.on("schemaReceived", (schema: ModelSchema) => {});      // fires once, after the auto-fetch on "ready"
reactor.on("capabilitiesReceived", (capabilities: Capabilities) => {});
reactor.on("error", (error: ReactorError) => {});
reactor.on("statsUpdate", (stats: ConnectionStats) => {});      // every 2s while ready
```

`ReactorMessage = { type: string; data: unknown }`.

Register listeners **before** `connect()`. The `statusChanged` event fires during the connection sequence — late registration misses `waiting` and `ready` transitions.

## Publishing a webcam (video-to-video)

The client doesn't declare inputs ahead of time; just publish once the session is `ready`. The track name must match the model's input attribute exactly.

```ts
import { Reactor } from "@reactor-team/js-sdk";

const reactor = new Reactor({ modelName: "helios" });

const stream = await navigator.mediaDevices.getUserMedia({ video: { width: 512, height: 512 } });
const [track] = stream.getVideoTracks();

reactor.on("statusChanged", async (status) => {
  if (status === "ready") {
    await reactor.publishTrack("webcam", track); // "webcam" = model's input attribute name
  }
});

await reactor.connect(jwt);

// Teardown
window.addEventListener("beforeunload", async () => {
  await reactor.unpublishTrack("webcam"); // never rejects — safe here
  stream.getTracks().forEach((t) => t.stop());
  await reactor.disconnect();
});
```

A name mismatch produces silent failure — no error, no frames. If video-to-video produces no output, check this first.

## File uploads

Commands that take files upload first and pass the returned `FileRef` into `sendCommand`. `FileRef` values mix freely with scalar args in the same command payload, and multiple `FileRef` values can be passed in one command — but only **top-level** values in `data` are detected and extracted, not nested ones:

```ts
const ref = await reactor.uploadFile(file);                    // File | Blob
const refWithName = await reactor.uploadFile(blob, { name: "photo.jpg" });

await reactor.sendCommand("set_image", { image: ref, strength: 0.7 });
```

## Recording and clips

New in 3.0.0 — no 2.x equivalent. `requestClip()`/`requestRecording()` throw normally (unlike `sendCommand`, see below), so wrap them in try/catch:

```ts
import { DEFAULT_PLAYLIST_POLL_SLACK_MS, downloadClipAsFile, fetchPlaylist } from "@reactor-team/js-sdk";

const clip = await reactor.requestClip(5); // last 5 seconds; or reactor.requestRecording() for the whole session

// downloadClipAsFile() itself polls the manifest with no limit by default.
// Bound the first poll explicitly if you want a hard timeout:
const controller = new AbortController();
await fetchPlaylist(clip.playlistUrl, {
  predictedReadyAtMs: clip.predictedReadyAtMs,
  slackMs: DEFAULT_PLAYLIST_POLL_SLACK_MS,
  signal: controller.signal,
  jwt,
});

const blob = await reactor.downloadClipAsFile(clip, "clip.mp4", {
  jwt,   // downloadClipAsFile does NOT inherit the Reactor instance's own JWT
  signal: controller.signal,
  onProgress: ({ fetched, total }) => console.log(`fetched ${fetched}/${total}`),
});
```

`Clip = { sessionId, kind: "snap" | "recording", startMarker, endMarker, nowMarker, predictedReadyAtMs, playlistUrl }`. The standalone `downloadClipAsFile`/`fetchPlaylist`/`DEFAULT_PLAYLIST_POLL_SLACK_MS` exports work with no `Reactor` instance at all — useful for downloading a `Clip` you received via a message or stored elsewhere. React apps get `ClipPlayer`/`ClipDownloadButton`/`useClipDownload` components for this — see [react.md](react.md).

## Reading command replies

`sendCommand()` now genuinely awaits `reactor-core`'s correlated reply (bounded by `controlRequestTimeoutMs`, not infinite) and resolves with it — this is the single biggest behavior change from 2.x, where it resolved to `void` almost immediately without waiting for anything:

```ts
const reply = await reactor.sendCommand("list_snapshots");
const snapshots = (reply?.data as { snapshots?: Snapshot[] } | undefined)?.snapshots ?? [];
```

`reply` is `undefined` if the handler acknowledged with no message body (common for simple `set_<field>` setters) — always guard with `?.`. Discover a model's actual command surface via the `schemaReceived` event / `getSchema()` rather than assuming: `Object.keys(schema.paths ?? {})`.

`sendCommand()`'s `data` accepts any object except a function — not strictly `Record<string, unknown>` — so a codegen'd or hand-written params `interface` can be passed directly with no `as Record<string, unknown>` cast.

**`sendCommand()` never rejects.** Failure routes through `getLastError()` / the `error` event and resolves `undefined` — this is the one call on `Reactor` that doesn't throw on failure. Every other call (`connect`, `publishTrack`, `uploadFile`, `requestClip`, ...) throws/rejects normally.

## Error handling

The SDK emits `ReactorError` via the `"error"` event, and the same class is what a rejected call throws — one hierarchy, not two shapes that can disagree. It's a real `class`, not a flat interface:

```ts
class ReactorError extends Error {
  readonly code: string;
  readonly message: string;
  readonly recoverable: boolean;
  readonly status: number | undefined;        // HTTP status, when applicable
  readonly operation: string | undefined;      // which call failed, e.g. "connect"
  readonly retry_after_ms: number | undefined;
  readonly timestamp_ms: number;
  readonly retryAfter: number | undefined;     // compat alias for retry_after_ms
  readonly timestamp: number;                  // compat alias for timestamp_ms
}
```

**There is no `component` field** — 2.x's `"api" | "gpu"` split is gone entirely; `reactor-core` never reported it, 2.x populated it locally per call site.

Named subclasses, each just a fixed `code`, all exported from the package: `NetworkError`, `UnauthorizedError` (401/403), `NotFoundError` (404), `ConflictError` (409), `RateLimitedError` (429), `BadRequestError` (other 4xx), `ServerError` (5xx), `VersionMismatchError`, `DecodeError`, `InvalidStateError`, `SessionTerminalError`, `MessageTooLargeError`, `TransportError`, `DisconnectedError`, `RequestTimeoutError`, `AbortedError`, `RecorderDisabledError`.

Prefer `instanceof` over string-matching `error.code` — the code vocabulary is open-ended; an unrecognized code falls back to the base `ReactorError` class rather than throwing something unexpected:

```ts
import { UnauthorizedError } from "@reactor-team/js-sdk";

reactor.on("error", (error) => {
  if (error instanceof UnauthorizedError) {
    // bad/expired JWT — re-mint and reconnect, don't just retry
    return;
  }
  if (error.recoverable) {
    setTimeout(() => reactor.reconnect(), (error.retryAfter ?? 3) * 1000);
    return;
  }
  console.error(`${error.code}: ${error.message}`);
});
```

Wrap `connect()` and any `sendCommand()`-adjacent throwing call (`publishTrack`, `uploadFile`, `requestClip`, `requestRecording`, `pauseTrack`/`resumeTrack`) in try/catch — remember `sendCommand()` itself is the one exception that never throws.

## Env vars

| Context | Name |
|---|---|
| Node server minting JWTs | `REACTOR_API_KEY` |
| Browser bundle | **do not expose** — use a server route (see [authentication.md](authentication.md)) |

`NEXT_PUBLIC_*` / `VITE_*` variables are inlined into the shipped JS; never use them for production API keys.
