# Reactor models, commands, and messages

## Commands vs. messages

Reactor sessions carry two kinds of non-video data alongside the video stream:

- **Commands** — named actions your app sends to the model. Change what the model is doing in real time.
- **Messages** — structured JSON the model sends back to your app. Report state, signal events, push any data the model wants to share.

```ts
// JS
await reactor.sendCommand("set_prompt", { prompt: "a forest at dawn" });

reactor.on("message", (msg) => {
  if (msg.type === "state") console.log("frame:", msg.data.current_frame);
});
```

```python
# Python
await reactor.send_command("set_prompt", {"prompt": "a forest at dawn"})

@reactor.on_message
def on_message(msg):
    if msg.get("type") == "state":
        print("frame:", msg["data"]["current_frame"])
```

Commands can only be sent when the connection is `ready`. Messages can arrive at any time during the session.

## Authoritative source

Every model defines its own command and message surface. The authoritative spec for any model is its docs page:

- **Models index:** https://docs.reactor.inc/model-api-reference/overview
- **Per-model pages:** `https://docs.reactor.inc/model-api-reference/<name>/overview` (plus `/schema`, `/prompt-guide`, and `/tutorial` under the same path)

Each model page lists:
- Accepted commands (name, parameters, types)
- Messages the model emits
- Declared input tracks (client `publishTrack` / `publish_track` target)
- Declared output tracks (client `receive` side — read via `tracks[name]` in JS, `reactor.tracks` / `reactor.track(name)` in Python)

Fetch the model page before writing a new integration. Command surfaces drift — do not assume commands from prior integrations still apply.

## Currently shipping

- **Helios** — interactive real-time video generation with autoregressive chunked diffusion and image-to-video support. Model slug: `helios`. See the `helios-prompts` skill for prompt-authoring rules.
- **LingBot** — real-time navigable video model with WASD movement, look controls, and live prompt steering. Model slug: `lingbot`. See the `lingbot-world` skill for world-authoring rules.
- **LingBot World 2** — image-anchored navigable environments with two-axis WASD driving, directed camera control, and live prompt steering. Model slug: `lingbot-world-2`. See the `lingbot-world-2-prompts` skill.
- **SANA-Streaming** — real-time streaming video editing: surgical text-driven edits to uploaded clips or a live webcam feed. Model slug: `sana-streaming`. See the `sana-streaming-prompts` skill for instruction-authoring rules.
- **HappyOyster** — permanent explorable worlds built from a prompt: play with held controls or direct with text instructions. Model slug: `happy-oyster`.
- **X2** — video transformation with character, clothing, style, and trajectory control via reference guidance. Model slug: `x2`.
- **LongLive-2.0** — autoregressive multi-shot video direction, soft transitions and hard cuts, live or on schedule. Model slug: `longlive-v2`.
- **LTX** — see its own docs page for current details; too new for this skill to have specifics verified yet.

Model slugs drift and new models ship regularly — **always take the slug from the model's own docs page**, not from this list or from a prior integration. Check `https://docs.reactor.inc/model-api-reference/overview` for the current roster before writing code against anything else.

## Scaffolding a new app

`npx create-reactor-app my-app --model=<slug>` scaffolds a starter project wired to a specific model. Faster than hand-assembling provider/hooks boilerplate for a new integration — reach for it before writing a app from scratch.

## Typed per-model SDKs

Beyond the base `@reactor-team/js-sdk` (untyped commands/messages — you pass plain objects and verify shapes against the docs yourself), TypeScript also gets **typed, per-model** packages published as `@reactor-models/<model>`, generated from the model's schema. These give you compile-time checked command params and message types instead of the base SDK's `Record<string, unknown>`-shaped `sendCommand`/`message`. Python has no equivalent — it stays on the base `reactor-sdk` package regardless of model.

Use a typed package when working with one specific, known model long-term; use the base SDK (per [javascript.md](javascript.md) / [python.md](python.md)) for anything generic, exploratory, or multi-model.

## Track naming contract

For any video-to-video model, three names must agree exactly:

1. The publish call: JS `reactor.publishTrack("webcam", track)` / Python `reactor.publish_track("webcam", track)`
2. The model's input attribute as declared in its capabilities (advertised by the server)
3. The identifier used in any related command parameters

Mismatch at any layer causes silent failure — no error, no frames. If a video-to-video integration produces no output, check this first.

## Writing code for a new model

1. Fetch the model's docs page from `https://docs.reactor.inc/model-api-reference/<name>/overview`.
2. Identify its input/output tracks. Match names exactly when publishing.
3. Identify its command surface. Verify every command name and argument shape against the docs before sending.
4. Check for model-specific auth, rate limits, or session constraints.
5. Smoke-test: connect → wait for `ready` → send one command → verify a frame arrives. Layer real logic only after that round trip works.

## When the docs disagree with the SDK

If a command shape differs between the docs page and what the SDK accepts, trust the SDK error message (it comes from the live model) and flag the docs discrepancy to the Reactor team.
