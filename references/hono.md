# Hono Setup

## When to use this reference

- User has `new Hono()` or `hono/` imports
- App uses `app.get`, `app.post`, `app.use` pattern
- Common on Cloudflare Workers, Node.js, Bun, Deno

## Install

```bash
npm install @flarelog/sdk hono
```

## Setup

### Zero-config (recommended — auto-detects runtime)

```typescript
import { Hono } from "hono";
import { honoMiddleware } from "@flarelog/sdk/hono";

const app = new Hono();
app.use("*", honoMiddleware());

app.get("/api/hello", (c) => {
  c.get("logger").info("Hello!");
  return c.json({ ok: true });
});

export default app;
```

The zero-arg `honoMiddleware()` auto-resolves `FLARELOG_API_KEY` from:
1. `c.env` — Hono's request context (Workers binding)
2. `process.env` — Node, Bun, Deno

### Eager logger (Node/Bun/Deno)

```typescript
import { Hono } from "hono";
import { flarelog } from "@flarelog/sdk";
import { honoMiddleware } from "@flarelog/sdk/hono";

const logger = flarelog({
  apiKey: process.env.FLARELOG_API_KEY!,
  sampleRate: 0.1,
});

const app = new Hono();
app.use("*", honoMiddleware(logger));
```

### Factory (full custom control — multi-tenant, custom env)

```typescript
import { Hono } from "hono";
import { flarelog } from "@flarelog/sdk";
import { honoMiddleware } from "@flarelog/sdk/hono";

const app = new Hono();
app.use("*", honoMiddleware((c) => {
  // c.env is available per-request on Workers
  return flarelog({ apiKey: c.env.TENANT_KEY, workerMode: true });
}));
```

## Using the logger in handlers

The middleware attaches the logger to `c.get("logger")`:

```typescript
app.get("/api/users/:id", (c) => {
  const logger = c.get("logger");
  logger.info("Fetching user", { userId: c.req.param("id") });
  return c.json({ id: c.req.param("id") });
});
```

The child logger includes `{ source: "hono", traceId, method, path }`.

## Hono on Cloudflare Workers

Hono runs natively on Workers. The zero-arg form handles `env` binding
resolution automatically. For best results, also deploy the Tail Worker (see
`references/tail-worker.md`).

```typescript
export default {
  fetch: app.fetch,
};
```

## What the middleware does

- Extracts traceId from `x-trace-id` header or `crypto.randomUUID()`
- Creates a child logger with request context
- Attaches `c.get("logger")` and `c.set("traceId", ...)`
- Logs "Request completed" with duration and status
- Calls `logger.flush()` after each request (critical on Workers)
- Captures thrown errors, logs them, then re-throws

## See also

- `references/env-resolution.md` — env var matrix
- `references/tail-worker.md` — crash capture on Workers
- `references/cloudflare-workers.md` — raw Workers setup
