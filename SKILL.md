---
name: flarelog
description: >-
  Add FlareLog logging, error monitoring, and observability to any JavaScript app.
  Handles Cloudflare Workers, Vercel, Lovable, TanStack Start, Next.js, Hono, Express,
  React, and plain Node.js. Use this skill whenever the user mentions FlareLog,
  flarelog, @flarelog/sdk, error logging, observability, crash capture, Tail Worker,
  cost burn alerts, or wants to add logging/monitoring to their app — even if they
  don't name FlareLog explicitly. Also use when the user is deploying a Lovable,
  Cursor, or AI-generated app and needs production visibility.
---

# FlareLog Integration Skill

You are helping the user integrate FlareLog into their application. FlareLog is a
zero-dependency observability SDK that ships logs, errors, and W3C traces from any
JavaScript runtime to a FlareLog dashboard or any OTLP backend.

## The two-layer architecture (understand this first)

FlareLog has **two complementary layers**. Most users need both:

1. **The SDK** (`@flarelog/sdk`) — runs *inside* the user's app. Captures errors
   thrown during request handling, console.log output, and unhandled rejections.
   Installed via `npm install @flarelog/sdk`. Framework-specific wrappers live at
   subpaths like `@flarelog/sdk/tanstack-start`, `@flarelog/sdk/hono`, etc.

2. **The Tail Worker** (Cloudflare Workers only) — runs *outside* the user's app,
   after each request finishes. Captures crashes the SDK can't see: CPU timeouts,
   memory exhaustion (Error 1027), startup failures, and Error 1101. Deployed as
   a separate Worker from the [tail-worker template](https://github.com/flarelog-dev/tail-worker).

If the user is on Cloudflare Workers (including Lovable), they need **both** the
SDK and the Tail Worker. The SDK alone misses the most damaging crashes. Make sure
to explain this — it's the #1 reason users think "FlareLog doesn't work" when
they only installed the SDK.

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

If the user is on Cloudflare Workers (any framework), also read
`references/tail-worker.md` — it covers the Tail Worker setup that catches
crashes the SDK misses.

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

### 3d. Deploy the Tail Worker (Cloudflare Workers only)

If the user is on Cloudflare Workers or Lovable, the SDK alone is not enough —
it can't see crashes that happen before the code runs (CPU timeout, OOM, startup
failure, Error 1101). They need the Tail Worker.

Walk them through:
1. Clone `https://github.com/flarelog-dev/tail-worker`
2. `wrangler secret put FLARELOG_API_KEY`
3. Edit `wrangler.toml` to point at their app's Worker name
4. `npm run deploy`

See `references/tail-worker.md` for the full instructions and pitfalls.

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

4. **Forgetting the Tail Worker** — the SDK alone misses Error 1101, CPU
   timeouts, and OOM crashes. On Workers, always set up the Tail Worker too.

5. **`VITE_` prefix on Lovable** — don't prefix `FLARELOG_API_KEY` with `VITE_`.
   That pushes the key into the client bundle and leaks it. Server-only secrets
   (no prefix) are injected as Worker bindings.

6. **Sentry conflict** — if the user has Sentry installed, FlareLog and Sentry
   can coexist. Don't tell them to remove Sentry. FlareLog catches crashes Sentry
   can't see; Sentry has session replay FlareLog doesn't. They complement each other.

## Tone and audience

Many FlareLog users are "vibe coders" — they built their app with Lovable, Cursor,
v0, or Bolt, and they're not experienced developers. They don't know what
"observability" or "OTel" means. When you detect this audience:

- Use plain English, not jargon
- Explain *why* each step matters (not just *what* to do)
- Emphasize the 5-minute setup promise
- Don't assume they know what a "Worker" or "binding" is — explain briefly
- Offer to walk through each step interactively

For experienced developers (they mention wrangler, TypeScript, CI/CD), you can
be more technical and skip the hand-holding.

## What NOT to do

- Don't invent API signatures — if you're not sure, read the reference file
- Don't tell users to use `getRequestEvent()` — it doesn't exist in v1 stable
- Don't create loggers at module scope on Cloudflare Workers
- Don't tell users to remove Sentry (they coexist fine)
- Don't skip the Tail Worker on Cloudflare Workers — it's not optional for crash capture
- Don't prefix env vars with `VITE_` on Lovable
