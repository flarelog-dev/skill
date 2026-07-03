---
name: flarelog
description: >-
  Add FlareLog logging, error monitoring, and observability to any JavaScript app.
  Handles Cloudflare Workers, Vercel, Lovable, TanStack Start, Next.js, Hono, Express,
  React, and plain Node.js. Use this skill whenever the user mentions FlareLog,
  flarelog, @flarelog/sdk, error logging, observability, crash capture,
  cost burn alerts, or wants to add logging/monitoring to their app — even if they
  don't name FlareLog explicitly. Also use when the user is deploying a Lovable,
  Cursor, v0, Bolt, or AI-generated app and needs production visibility.
---

# FlareLog Integration Skill

You are helping the user integrate FlareLog into their application. FlareLog is a
zero-dependency observability SDK that ships logs, errors, and W3C traces from any
JavaScript runtime to a FlareLog dashboard or any OTLP backend.

## The two-layer architecture (understand this first)

For almost every app, **the SDK (`@flarelog/sdk`) is all you need**. It runs
*inside* the user's app. Captures errors thrown during request handling,
console.log output, and unhandled rejections. Installed via
`npm install @flarelog/sdk`. Framework-specific wrappers live at subpaths like
`@flarelog/sdk/tanstack-start`, `@flarelog/sdk/hono`, etc.

There is also an optional **Tail Worker** layer that only applies to **raw
Cloudflare Workers** (not Lovable, not TanStack Start, not Vercel, not Node).
It runs *outside* the app and catches crashes the SDK can't see: CPU timeouts,
memory exhaustion, startup failures, and Error 1101. It is deployed from the
[tail-worker template](https://github.com/flarelog-dev/tail-worker).

If the user is on **Lovable or TanStack Start**, do **not** mention the Tail
Worker. The SDK middleware handles crash capture for them. Only bring up the
Tail Worker if the user explicitly says they are on a raw Cloudflare Worker and
wants that extra out-of-band coverage.

## Step 1: Identify the user's stack

Before writing any code, determine:

1. **Framework**: TanStack Start, Next.js, Hono, Express, React (browser), or plain handler?
2. **Platform**: Cloudflare Workers, Vercel, Node.js, or browser-only?
3. **AI tool** (if applicable): Lovable, Cursor, v0, Bolt? This affects how you phrase instructions — Lovable users edit via chat, Cursor users edit files directly.

Ask the user if it's not obvious from context. Common signals:

| Signal | Likely stack |
|--------|-------------|
| "Lovable", "lovable.app", "lovable.dev" | Lovable → TanStack Start on Cloudflare Workers |
| `createStart`, `@tanstack/react-start`, `src/start.ts` | TanStack Start |
| `next.config`, `pages/api`, `app/api` | Next.js |
| `new Hono()`, `hono/` | Hono |
| `express()`, `app.use` | Express |
| `wrangler.toml`, `wrangler.jsonc`, `export default { fetch }` | Raw Cloudflare Workers |
| `vercel.json`, `api/` directory, `runtime: "edge"` | Vercel Serverless/Edge |
| `useEffect`, `useState`, `.tsx` in browser context | React/browser |

## Step 2: Read the matching reference file

Based on the stack, read **only the relevant reference file** before writing code.
Each reference file has the exact install command, code snippet, and pitfalls for
that framework.

| Stack | Reference file |
|-------|---------------|
| TanStack Start / Lovable | `references/tanstack-start.md` |
| Next.js (Pages or App Router) | `references/nextjs.md` |
| Hono | `references/hono.md` |
| Express | `references/express.md` |
| Raw Cloudflare Workers | `references/cloudflare-workers.md` |
| Vercel (Serverless / Edge) | `references/vercel.md` |
| React / Browser | `references/react.md` |

If the user is on **raw Cloudflare Workers** and explicitly wants out-of-band
crash coverage, read `references/tail-worker.md`. For every other stack —
including Lovable and TanStack Start — skip the Tail Worker entirely.

## Step 3: Walk the user through setup

Follow this sequence. Adapt the phrasing to the user's experience level — if they
mention Lovable or "I'm not a developer", use simpler language and avoid DevOps
jargon. If they mention Cursor/wrangler/TypeScript, you can be more technical.

### 3a. Install the SDK

```bash
npm install @flarelog/sdk
```

Framework peer deps (e.g. `@tanstack/react-start`) are optional — only install
if the user doesn't already have them.

### 3b. Add the middleware/wrapper

Use the exact code from the reference file. **Do not improvise** — the SDK has
specific patterns for each framework, and deviating breaks things. Common
pitfalls are called out in each reference file; read them.

### 3c. Set the API key

The user needs a `FLARELOG_API_KEY`. They get it from the FlareLog dashboard at
[flarelog.dev/login](https://flarelog.dev/login) (free, no credit card).

**Where to set it depends on the platform:**

| Platform | Where to set the key |
|----------|---------------------|
| Node.js / Vercel | `.env` file or dashboard env vars — `process.env.FLARELOG_API_KEY` works at module load |
| Cloudflare Workers (with `nodejs_compat`) | `wrangler.jsonc` `[vars]` or dashboard secrets — `process.env` works per-request inside `.server()` callbacks |
| Cloudflare Workers (without `nodejs_compat`) | Dashboard secrets only — read via `cloudflare:workers` `env` binding. The zero-arg middleware handles this automatically. |
| Lovable | Lovable's "Secrets / Environment variables" panel (do NOT prefix with `VITE_`) |

**Critical for Cloudflare Workers / Lovable:** Do NOT create the logger at module
scope with `flarelog({ apiKey: env.FLARELOG_API_KEY })`. The `env` binding is
`undefined` at module load. Use the zero-arg middleware form
(`tanstackStartMiddleware()` / `honoMiddleware()`) which reads the binding
lazily on the first request. This is the #1 cause of "logs work in dev but not
in preview" — see `references/env-resolution.md` for the full matrix.

### 3d. Optional: Tail Worker for raw Cloudflare Workers (Lovable users skip this)

For Lovable, TanStack Start, Next.js, Hono on Workers, Vercel, Express, React,
and Node, the SDK is enough. **Do not mention the Tail Worker to these users.**

Only mention the Tail Worker if the user is on a **raw Cloudflare Worker** and
explicitly asks for out-of-band crash capture (CPU timeouts, OOM, startup
failures, Error 1101). Even then, present it as an optional advanced add-on.
See `references/tail-worker.md`.

### 3e. (Optional) Connect the MCP server

If the user uses Cursor or Claude Desktop, offer to set up the MCP server so
their AI can read production logs and debug for them. See
`references/mcp-setup.md`.

## Step 4: Verify it works

After setup, tell the user to:
1. Deploy their app
2. Hit a few endpoints (or visit the site)
3. Check the FlareLog dashboard — they should see logs within a few seconds

If logs don't appear:
- On Workers: check that the API key is set as a secret (not a `[vars]` plaintext
  var) and that `nodejs_compat` is enabled OR they're using the zero-arg middleware
- On Node/Vercel: check that `.env` is loaded and `FLARELOG_API_KEY` is set
- Check the browser console for `[FlareLog] No backend configured` — this means
  the SDK couldn't find the API key and fell back to console-only

## Common pitfalls (read these — they bite everyone)

1. **Module-scope `env.FLARELOG_API_KEY` on Workers** — `env` is undefined at
   module load. Use the zero-arg middleware or create the logger inside the
   handler. See `references/env-resolution.md`.

2. **`getRequestEvent` from `@tanstack/react-start`** — this API was removed
   before v1 stable. Do NOT use it. The SDK's zero-arg middleware uses
   `cloudflare:workers` `env` binding instead.

3. **`import("cloudflare:workers")` in source** — bundlers (Vite, esbuild,
   webpack) can't resolve this runtime-only module. The SDK hides it behind
   `new Function("return import(spec)")` so it's invisible to static analysis.
   If you're writing custom code that reads the binding, use the same pattern.

4. **Telling Lovable users they need a Tail Worker** — they don't. The SDK
   middleware catches request errors and ships them to FlareLog. Mentioning a
   separate Tail Worker will confuse vibe coders and isn't required.

5. **`VITE_` prefix on Lovable** — don't prefix `FLARELOG_API_KEY` with `VITE_`.
   That pushes the key into the client bundle and leaks it. Server-only secrets
   (no prefix) are injected as Worker bindings.

6. **Sentry conflict** — if the user has Sentry installed, FlareLog and Sentry
   can coexist. Don't tell them to remove Sentry. FlareLog catches crashes Sentry
   can't see; Sentry has session replay FlareLog doesn't. They complement each other.

## Tone and audience

Many FlareLog users are "vibe coders" — they built their app with Lovable,
Cursor, v0, or Bolt and they're not experienced developers. They don't know what
"observability" or "OTel" means, and they don't edit Wrangler configs.

When you detect this audience (especially Lovable):

- Use plain English, not jargon. Say "logs and error tracking" instead of
  "observability." Say "Your app's dashboard" instead of "Worker bindings."
- Explain *why* a step matters in one sentence, then give the exact code or
  chat prompt.
- Give them copy/paste blocks they can drop into the Lovable chat.
- Emphasize the "5-minute setup" promise.
- Don't assume they know what a "Worker", "middleware", or "env binding" is —
  name the file (`src/start.ts`) and the line they need to add.
- Keep instructions short. One ask per message is better than a wall of text.
- Offer to walk them through it step by step.

For experienced developers (they mention wrangler, TypeScript, CI/CD), you can
be more technical and skip the hand-holding.

## What NOT to do

- Don't invent API signatures — if you're not sure, read the reference file
- Don't tell users to use `getRequestEvent()` — it doesn't exist in v1 stable
- Don't create loggers at module scope on Cloudflare Workers
- Don't tell users to remove Sentry (they coexist fine)
- Don't mention the Tail Worker to Lovable, TanStack Start, or other framework
  users — the SDK is all they need
- Don't prefix env vars with `VITE_` on Lovable
