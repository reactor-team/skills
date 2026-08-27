# Reactor React SDK

Declarative React bindings built on top of `@reactor-team/js-sdk`. For imperative JS usage see [javascript.md](javascript.md).

This reference documents `@reactor-team/js-sdk` **3.0.0** — see [javascript.md](javascript.md) for what's new versus the 2.x line.

## Install

```bash
npm install @reactor-team/js-sdk react react-dom
```

`react` is a **peer dependency** (`^18.0.0 || ^19.0.0`) — install it yourself. React hooks and components ship from the same top-level package; there is no `/react` subpath.

## Provider setup

Wrap the subtree that needs Reactor access. Tracks are advertised by the server from the model's capabilities — do **not** pass `receive` or `send` props.

```tsx
import { ReactorProvider, ReactorView } from "@reactor-team/js-sdk";

export function App({ jwtToken }: { jwtToken: string }) {
  return (
    <ReactorProvider
      modelName="helios"
      jwtToken={jwtToken}
      connectOptions={{ autoConnect: true }}
    >
      <ReactorView className="w-full aspect-video" />
    </ReactorProvider>
  );
}
```

`ReactorProviderProps`: `apiUrl?`, `modelName` (required), `local?`, `modelTracks?` (`TrackCapability[]`, matches the vanilla `Reactor` constructor), `jwtToken?` (`JwtSource` — string or a `() => string | Promise<string>` resolver), `connectOptions?` (`ConnectOptions & { autoConnect?: boolean }`, default `autoConnect: false`).

**Every one of those props is live.** Changing `apiUrl`/`modelName`/`local`/`modelTracks`/`jwtToken`/`connectOptions` tears down the current `Reactor` instance (`disconnect()` then dispose) and builds a fresh one — there's no in-place reconnect on a prop change. `modelTracks` and `connectOptions` are compared by value (`JSON.stringify`), so an inline object literal is fine and won't cause spurious rebuilds by itself — but a `jwtToken` resolver function recreated on every render (an inline arrow function, for instance) is compared by reference and **will** cause repeated rebuilds. Hoist it or wrap in `useCallback`.

Hooks below must be used inside a `ReactorProvider`.

## Hooks

```tsx
import {
  useReactor,
  useReactorMessage,
  useReactorInternalMessage,
  useStats,
} from "@reactor-team/js-sdk";
```

### `useReactor(selector)`

Shallow-equality-checked selector over the store state. Destructure the fields you need:

```tsx
function StatusBanner() {
  const { status, lastError } = useReactor((s) => s);
  return <div>{status}{lastError ? ` — ${lastError.message}` : ""}</div>;
}
```

Store state fields:

| Field | Type |
|---|---|
| `status` | `"disconnected" \| "connecting" \| "waiting" \| "ready"` |
| `sessionId` | `string \| undefined` |
| `lastError` | `ReactorError \| undefined` — see [javascript.md](javascript.md#error-handling) for the class hierarchy |
| `lastMessage` | `ReactorMessage \| undefined` — most recent **application-scope** `message` only; `runtimeMessage` is not mirrored here |
| `tracks` | `Record<string, MediaStreamTrack>` — model-emitted tracks keyed by name; reset to `{}` on `"disconnected"` |
| `jwtToken` | `JwtSource \| undefined` — the value the provider was created with; not resynced on prop change beyond a rebuild |
| `connectOptions` | `ConnectOptions \| undefined` — the default options set at store creation |

Store action fields:

| Field | Signature |
|---|---|
| `connect` | `(jwt?, options?: ConnectOptions) => Promise<void>` |
| `disconnect` | `(recoverable?: boolean) => Promise<void>` |
| `reconnect` | `(options?: ConnectOptions) => Promise<void>` |
| `sendCommand` | `(command, data?, scope?) => Promise<ReactorMessage \| undefined>` — see [javascript.md](javascript.md#reading-command-replies), **never rejects** |
| `publish` | `(name, track: MediaStreamTrack) => Promise<void>` |
| `unpublish` | `(name) => Promise<void>` |
| `pauseTrack` | `(name) => Promise<void>` |
| `resumeTrack` | `(name) => Promise<void>` |
| `uploadFile` | `(file: File \| Blob, options?: { name?: string }) => Promise<FileRef>` |
| `requestClip` | `(durationSeconds: number) => Promise<Clip>` |
| `requestRecording` | `() => Promise<Clip>` |
| `downloadClipAsFile` | `(clip, filename?, options?) => Promise<Blob>` |

**Note the rename:** the store exposes `publish` / `unpublish` (not `publishTrack` / `unpublishTrack` — those names exist on the underlying `Reactor` class, but hook consumers use the shorter names).

Not mirrored in store state at all: schema, capabilities, stats. Reach the raw instance via `useReactor((s) => s.internal.reactor)` for those (`reactor.getSchema()`, `reactor.getCapabilities()`, `reactor.getStats()`) — or use `useStats()` below for stats specifically.

### `useReactorMessage(handler)`

Subscribes to model application messages. Handler is registered on mount, removed on unmount.

```tsx
function FrameCounter() {
  const [frame, setFrame] = useState(0);
  useReactorMessage((msg) => {
    if (msg.type === "state") setFrame((msg.data as { current_frame: number }).current_frame);
  });
  return <div>Frame: {frame}</div>;
}
```

### `useReactorInternalMessage(handler)`

Subscribes to platform-level (`runtimeMessage`) events — moderation, clip/recording lifecycle, and other control-plane data. Advanced; most apps use `useReactorMessage` instead.

### `useStats()`

Returns the latest `ConnectionStats` (RTT, jitter, bitrate, frames/sec, connection timings). Updates every ~2s while connected; resets to `undefined` on unmount or when the underlying `Reactor` instance is rebuilt.

## Rendering video

```tsx
import { ReactorView } from "@reactor-team/js-sdk";

function Scene() {
  return (
    <ReactorView
      className="w-full aspect-video rounded-lg"
      videoObjectFit="cover"
    />
  );
}
```

`ReactorViewProps`:

| Prop | Default | Purpose |
|---|---|---|
| `track` | `"main_video"` | Name of the recvonly video track to render |
| `audioTrack` | — | Optional recvonly audio track name (mixed into the same element) |
| `width`, `height` | — | Dimensions |
| `className`, `style` | — | Standard |
| `videoObjectFit` | `"contain"` | CSS `object-fit` |
| `muted` | `true` if no `audioTrack`, else `false` | Browser autoplay policies require muted-by-default |

For custom rendering or multiple named tracks, read `tracks[name]` from the store directly.

## Publishing a webcam

```tsx
import { ReactorProvider, WebcamStream, ReactorView } from "@reactor-team/js-sdk";

<ReactorProvider modelName="helios" jwtToken={jwt} connectOptions={{ autoConnect: true }}>
  <WebcamStream track="webcam" className="w-48 aspect-video" />
  <ReactorView className="w-full aspect-video" />
</ReactorProvider>;
```

`WebcamStream` handles `getUserMedia`, waits for `ready`, publishes the track, and unpublishes + stops the capture on unmount.

`WebcamStreamProps`:

| Prop | Default | Purpose |
|---|---|---|
| `track` | **required** | Sendonly video track name; must match the model's input attribute |
| `audio` | `false` | `boolean` or `MediaTrackConstraints` — capture audio alongside video |
| `audioTrack` | — | Sendonly audio track name; ignored unless `audio` is set |
| `videoConstraints` | `{ width: { ideal: 1280 }, height: { ideal: 720 } }` | Read **once at mount** — changing it later doesn't re-request the camera |
| `showWebcam` | `true` | Show the local preview |
| `className`, `style` | — | Standard |
| `videoObjectFit` | `"contain"` | CSS `object-fit` on the preview |
| `onPermissionDenied` | — | Called if `getUserMedia` is denied |
| `onPublished` | — | Called once the track successfully publishes |
| `onError` | — | Called on any other capture/publish error |

## Sending commands from components

```tsx
function PromptBox() {
  const { sendCommand, status } = useReactor((s) => s);

  const onSubmit = async (text: string) => {
    if (status !== "ready") return;
    // Command arg shapes are model-specific — verify against docs.reactor.inc/model-api-reference/<model>/schema
    const reply = await sendCommand("set_prompt", { prompt: text });
    // reply is `ReactorMessage | undefined` — most setter-style commands reply with nothing
  };

  return <input disabled={status !== "ready"} onChange={(e) => onSubmit(e.target.value)} />;
}
```

Always gate sends on `status === "ready"`, or disable the control until ready — otherwise the call resolves with `undefined` and does nothing (`sendCommand` never throws — see [javascript.md](javascript.md#reading-command-replies)).

## File uploads

```tsx
function ImagePicker() {
  const { uploadFile, sendCommand, status } = useReactor((s) => s);

  const onFile = async (file: File) => {
    if (status !== "ready") return;
    const ref = await uploadFile(file);
    await sendCommand("set_image", { image: ref });
  };

  return <input type="file" onChange={(e) => e.target.files?.[0] && onFile(e.target.files[0])} />;
}
```

`FileRef` values can be mixed with scalar args in the same command payload, and multiple files can be passed in one command — but only top-level values in the payload are detected.

## Recording and clips

New in 3.0.0 — no 2.x equivalent, and no old-docs analogue at all.

```tsx
import { useReactor, ClipPlayer, ClipDownloadButton } from "@reactor-team/js-sdk";

function ClipControls() {
  const { requestClip, status } = useReactor((s) => s);
  const [clip, setClip] = useState<Clip>();

  return (
    <>
      <button
        disabled={status !== "ready"}
        onClick={async () => setClip(await requestClip(5))}
      >
        Save last 5s
      </button>
      {clip && (
        <>
          <ClipPlayer clip={clip} />
          <ClipDownloadButton clip={clip}>Download</ClipDownloadButton>
        </>
      )}
    </>
  );
}
```

- **`ClipPlayer`** — plays a `Clip` via `hls.js` (dynamically imported) wherever MSE exists, including iOS Safari 17.1+; falls back to assembling a flat MP4 in memory on older iOS. Doesn't require a `ReactorProvider` in its tree — pass `getJwt` explicitly outside one, or let it inherit the provider's JWT resolver inside one. Props: `clip` (required), `getJwt?`, `slackMs?` (bounded manifest wait), `autoPlay?` (default `true`), `muted?` (default `true`), `className?`, `style?`, `onError?`.
- **`ClipDownloadButton`** — wraps `useClipDownload`. Props: `clip` (required), `getJwt?`, `filename?` (default `"reactor-clip.mp4"`), `children?` (node or a render function taking `ClipDownloadState`), `className?`, `style?`, `disabled?`, `onSuccess?`, `onError?`.
- **`useClipDownload(clip, options?)`** → `{ state, download, reset }`. `state` is `{ kind: "idle" } | { kind: "downloading"; fetched; total } | { kind: "error"; message }`. `download()` never rejects — errors surface via `state`, not a thrown error.

## Error handling and automatic reconnect

```tsx
function ErrorBanner() {
  const lastError = useReactor((s) => s.lastError);
  if (!lastError) return null;
  return (
    <div>
      <strong>{lastError.code}</strong>: {lastError.message}
    </div>
  );
}

function useReconnectOnRecoverable() {
  const lastError = useReactor((s) => s.lastError);
  const reconnect = useReactor((s) => s.reconnect);

  useEffect(() => {
    if (lastError?.recoverable) {
      const t = setTimeout(() => reconnect(), (lastError.retryAfter ?? 3) * 1000);
      return () => clearTimeout(t);
    }
  }, [lastError, reconnect]);
}
```

`ReactorError` is a real class hierarchy in 3.0.0 (`UnauthorizedError`, `RateLimitedError`, `ConflictError`, ...), not a flat interface — see [javascript.md](javascript.md#error-handling) for the full list. Prefer `instanceof` over matching `lastError.code` strings where you can.

## Manual connection control

Defer connection until a user action:

```tsx
<ReactorProvider modelName="helios" jwtToken={jwt} connectOptions={{ autoConnect: false }}>
  <ConnectButton />
</ReactorProvider>;

function ConnectButton() {
  const { connect, disconnect, reconnect, status } = useReactor((s) => s);
  return (
    <div>
      <button onClick={() => connect()} disabled={status !== "disconnected"}>Start</button>
      <button onClick={() => disconnect()} disabled={status === "disconnected"}>Stop</button>
      <button onClick={() => reconnect()}>Reconnect</button>
    </div>
  );
}
```

For brief network blips, pass `true` to `disconnect` so `reconnect()` can resume the server-side session:

```tsx
await disconnect(true);
// ... network recovers ...
await reconnect();
```

## Other exports

- `useReactor`'s `internal.reactor` — direct access to the underlying `Reactor` instance for anything not mirrored into store state (schema, capabilities, raw event subscriptions).
- `DEFAULT_PLAYLIST_POLL_SLACK_MS`, `downloadClipAsFile`, `fetchPlaylist`, `parsePlaylist` — the standalone recording module, usable with no provider/instance at all (same exports as [javascript.md](javascript.md#recording-and-clips)).
- `FileRef`, `isFileRef` — re-exported from the base package.
- `normalizeJwtSource` — re-exported from the base package.

Not re-exported from the top-level package (internal/advanced): `useReactorStore` and `ReactorContext` from the provider module.

## Cleanup

`ReactorProvider` disconnects on unmount and on `beforeunload`. `WebcamStream` stops local tracks on unmount. For SPAs with long-lived trees, call `disconnect()` on route change if you want to release the GPU session early.
