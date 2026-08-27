# Reactor authentication

Reactor uses API keys (format `rk_...`) to mint short-lived JWTs. Browsers connect with the JWT, never the raw API key. Python runs server-side and passes the API key directly to the SDK.

## API key

Create a key in the [Reactor Dashboard](https://reactor.inc/dashboard) → user menu → **Account Settings** → **API Keys**. Keys start with `rk_`. Never commit one to source control.

## JavaScript / TypeScript

### Mint a token

`POST https://api.reactor.inc/tokens` with the API key in the `Reactor-API-Key` header. The response contains a JWT valid for **6 hours**.

```ts
const r = await fetch("https://api.reactor.inc/tokens", {
  method: "POST",
  headers: { "Reactor-API-Key": "rk_your_api_key_here" },
});
const { jwt } = await r.json();
```

There is **no** `fetchInsecureJwtToken` helper in the current SDK — call the endpoint directly.

### Production: server-side proxy

Do not ship `rk_...` keys to the browser. Expose a server route that mints the token:

```ts
// Next.js — app/api/token/route.ts
import { NextResponse } from "next/server";

export async function POST() {
  const r = await fetch("https://api.reactor.inc/tokens", {
    method: "POST",
    headers: { "Reactor-API-Key": process.env.REACTOR_API_KEY! },
  });
  const { jwt } = await r.json();
  return NextResponse.json({ jwt });
}
```

```ts
// Express
app.post("/api/token", async (_req, res) => {
  const r = await fetch("https://api.reactor.inc/tokens", {
    method: "POST",
    headers: { "Reactor-API-Key": process.env.REACTOR_API_KEY! },
  });
  const { jwt } = await r.json();
  res.json({ jwt });
});
```

### Frontend: consume the token

```tsx
"use client";

import { use } from "react";
import { ReactorProvider, ReactorView } from "@reactor-team/js-sdk";

async function getToken() {
  const r = await fetch("/api/token", { method: "POST" });
  const { jwt } = await r.json();
  return jwt;
}

const tokenPromise = getToken();

export default function App() {
  const token = use(tokenPromise);
  return (
    <ReactorProvider modelName="helios" jwtToken={token}>
      <ReactorView className="w-full aspect-video" />
    </ReactorProvider>
  );
}
```

Or in an imperative flow:

```ts
import { Reactor } from "@reactor-team/js-sdk";

const r = await fetch("/api/token", { method: "POST" });
const { jwt } = await r.json();

const reactor = new Reactor({ modelName: "helios" });
await reactor.connect(jwt);
```

### Token refresh

Tokens expire after 6 hours. For sessions approaching that limit, re-fetch `/api/token` and reconnect before the JWT expires, or in response to an `UnauthorizedError` (dispatch on `error instanceof UnauthorizedError` — see [javascript.md](javascript.md#error-handling); in 3.0.0 this is a real class, not a `component`/`code` pair to string-match).

## Python

Pass the API key as the constructor's second positional argument; the SDK exchanges it for a JWT internally on `connect()`:

```python
import os
from reactor_sdk import Reactor

reactor = Reactor("helios", os.environ["REACTOR_API_KEY"])

await reactor.connect()
```

Python runs server-side, so there is no browser exposure to worry about. Just keep the key in a secret manager or env var — never in source.

Alternatively, pass a JWT directly via the keyword-only `jwt=` argument (e.g. if some other part of your system already mints tokens) — it wins over `api_key` and the SDK never re-mints it:

```python
reactor = Reactor("helios", jwt=some_existing_jwt)
```

If the exchange itself fails (bad key, unreachable auth host, malformed response), the SDK raises `AuthError` — a plain `RuntimeError`, not a `ReactorError` — around `connect()`. Catch it separately from `ReactorError`/`@on_error`.

## Env var conventions

| Context | Name |
|---|---|
| Node / Python server | `REACTOR_API_KEY` |
| Browser bundles | **do not expose the key** — proxy through a server route |

`NEXT_PUBLIC_REACTOR_API_KEY` and `VITE_REACTOR_API_KEY` inline the value into the shipped client bundle. Use them only for throwaway local dev on single-user demos.

## Common failure modes

- **`UnauthorizedError` on connect** (JS) / **`UnauthorizedError`** (Python) — JWT expired, malformed, or the API key is invalid. Verify the key starts with `rk_` and that the `/tokens` call returned 200. In Python, a failed API-key-to-JWT exchange itself raises `AuthError` instead — a separate, non-`ReactorError` exception (see [python.md](python.md#error-handling)).
- **Works locally, fails in prod** — API key was bundled into the client via a `NEXT_PUBLIC_*` / `VITE_*` var. Move minting to a server route.
- **Intermittent auth failures on long sessions** — JWT hitting its 6-hour TTL. Add a refresh before expiry or on recoverable errors.
- **403 on your own `/api/token` route** — your server isn't authenticating the user before proxying; add auth in front of the mint call.
