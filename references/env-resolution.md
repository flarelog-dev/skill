# Env Resolution Matrix

## The #1 cause of "logs work in dev but not in production"

On Cloudflare Workers (including Lovable), `env` bindings are only available
inside the `fetch` handler — NOT at module scope. Code like this breaks:

```typescript
// ❌ BREAKS on Workers — env is undefined at module load
const logger = flarelog({ apiKey: env.FLARELOG_API_KEY });

export default {
  fetch: workerFetch(logger, async (request, env, ctx) => {
    return new Response("Hello");
  }),
};
```

This works fine in `wrangler dev` (Node.js) but fails in production because
`env` is `undefined` when the module loads. The logger gets `apiKey: undefined`,
silently falls back to console-only logging, and nothing ships to the dashboard.

## The full resolution matrix

FlareLog's `autoLogger()` and the zero-arg middleware resolve `FLARELOG_API_KEY`
from the first source that has it, in this exact order:

| Priority | Source | Where it works | How to set it |
|----------|--------|----------------|---------------|
| 1 (highest) | Explicit `env` arg to `autoLogger(env)` | Everywhere | Pass `c.env` (Hono), `context.env` (Pages), or your own record |
| 2 | `process.env.FLARELOG_API_KEY` | Node, Vercel, Cloudflare Workers **with `nodejs_compat`** | `.env` file (dev), dashboard env vars (Vercel), `[vars]` in `wrangler.jsonc` (Workers) |
| 3 | `cloudflare:workers` `env` binding | Cloudflare Workers **without `nodejs_compat`**, Lovable | `wrangler secret put` or dashboard secrets |
| 4 | Public env vars exposed by the frontend bundler | Browser / frontend apps | `VITE_FLARELOG_API_KEY`, `NEXT_PUBLIC_FLARELOG_API_KEY`, `REACT_APP_FLARELOG_API_KEY` |
| 5 (fallback) | Nothing found → console-only + `console.warn` | — | — |

## Concrete answers to common questions

### "Does it work with just `.env`?"

**Yes**, on Node dev, Vercel, and Workers with `nodejs_compat`. Put
`FLARELOG_API_KEY=fl_xxx` in `.env` and you're done.

### "Does it work with just Cloudflare secrets?"

**Yes**, on Workers (with or without `nodejs_compat`). Add `FLARELOG_API_KEY`
in the Cloudflare dashboard or `wrangler secret put FLARELOG_API_KEY`. The SDK
reads it via the `cloudflare:workers` `env` binding.

### "Does it work with both?"

**Yes**. Priority is: explicit arg > `process.env` > `cloudflare:workers` binding.
If both are set, `process.env` wins (checked first and cached).

### "What if neither is set?"

The SDK logs a `console.warn` once per logger instance:
```
[FlareLog] No backend configured — falling back to console-only logging.
```
And ships nothing. Silence it with `flarelog({ warnOnConsoleFallback: false })`.

## Platform-specific patterns

### Node.js / Vercel

`process.env` is available at module load. Safe to create the logger eagerly:

```typescript
const logger = flarelog({ apiKey: process.env.FLARELOG_API_KEY! });
```

### Browser / frontend apps (Vite, Next.js, CRA)

The same `FLARELOG_API_KEY` is used in the browser, but your build tool decides
whether the variable is exposed to the client bundle:

| Framework | Env var name | How to read it |
|-----------|--------------|----------------|
| Vite | `VITE_FLARELOG_API_KEY` | `import.meta.env.VITE_FLARELOG_API_KEY` |
| Next.js | `NEXT_PUBLIC_FLARELOG_API_KEY` | `process.env.NEXT_PUBLIC_FLARELOG_API_KEY` |
| Create React App | `REACT_APP_FLARELOG_API_KEY` | `process.env.REACT_APP_FLARELOG_API_KEY` |

```bash
# .env (Vite example)
VITE_FLARELOG_API_KEY=fl_your_key
```

```typescript
const logger = flarelog({
  apiKey: import.meta.env.VITE_FLARELOG_API_KEY,
});
```

### Cloudflare Workers with `nodejs_compat`

`process.env` is populated per-request inside middleware `.server()` callbacks
(by `@cloudflare/vite-plugin`). Safe to use the zero-arg middleware — it reads
`process.env` on the first request:

```typescript
// TanStack Start
export const startInstance = createStart(() => ({
  requestMiddleware: [tanstackStartMiddleware() as never],
}));
```

### Cloudflare Workers WITHOUT `nodejs_compat`

`process.env` is empty. The SDK falls back to the `cloudflare:workers` `env`
binding (read via a `new Function()` constructor that's invisible to bundlers).
The zero-arg middleware handles this automatically.

### Lovable

Lovable runs on Workers. Secrets are set in Lovable's "Secrets / Environment
variables" panel. Do NOT prefix with `VITE_` — that pushes the key to the
browser bundle. Use the zero-arg middleware form.

### Hono on Workers

Pass `c.env` explicitly — it's the fastest path:

```typescript
app.use("*", async (c, next) => {
  const logger = await autoLogger(c.env);
  c.set("logger", logger);
  await next();
});
```

Or use the zero-arg `honoMiddleware()` which auto-resolves.

## The `cloudflare:workers` import (for custom code)

If you're writing custom code that reads the Worker binding directly:

```typescript
// ✅ Safe — hidden from bundler static analysis
const dynamicImport = new Function("spec", "return import(spec)") as (s: string) => Promise<{ env?: Record<string, string | undefined> }>;
const mod = await dynamicImport("cloudflare:workers");
const env = mod.env;
```

Do NOT write `await import("cloudflare:workers")` directly — Vite, esbuild,
webpack, and rollup all fail to resolve it at build time because there's no
installable npm package. The `new Function()` pattern makes the specifier
invisible to bundler static analysis.

## Testing env resolution

The SDK exports test helpers for this:

```typescript
import { __resetAutoLoggerCache, __setCloudflareEnvForTests } from "@flarelog/sdk/frameworks/auto-logger";

beforeEach(() => {
  __resetAutoLoggerCache();
  __setCloudflareEnvForTests({ FLARELOG_API_KEY: "test-key" });
});
```

This seeds the cache that production code would populate via
`import { env } from "cloudflare:workers"`.

## See also

- `references/tanstack-start.md` — zero-arg middleware for Lovable/TanStack Start
- `references/cloudflare-workers.md` — raw Workers setup
- `references/hono.md` — Hono setup (passes `c.env` explicitly)
