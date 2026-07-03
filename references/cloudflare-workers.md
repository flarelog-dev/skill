# Cloudflare Workers (Raw) Setup

## When to use this reference

- User has `wrangler.toml` or `wrangler.jsonc`
- App uses `export default { fetch(request, env, ctx) { ... } }`
- No framework (plain Worker handler)

If the user is using Hono on Workers, use `references/hono.md` instead.
If they're using TanStack Start on Workers (Lovable), use `references/tanstack-start.md`.

## Install

```bash
npm install @flarelog/sdk
```

## Setup

### The critical pattern: create the logger INSIDE the handler

On Cloudflare Workers, `env` bindings are only available inside the `fetch`
handler — NOT at module scope. This is the #1 mistake users make.

**❌ Wrong (breaks on Workers):**
```typescript
import { flarelog, workerFetch } from "@flarelog/sdk";

const logger = flarelog({ apiKey: env.FLARELOG_API_KEY }); // env is undefined here!

export default {
  fetch: workerFetch(logger, async (request, env, ctx) => {
    return new Response("Hello");
  }),
};
```

**✅ Right (zero-arg form — auto-reads the binding):**
```typescript
import { flarelog, workerFetch } from "@flarelog/sdk";

export default {
  fetch: workerFetch(
    flarelog(), // auto-detects env on first request
    async (request, env, ctx) => {
      return new Response("Hello");
    },
  ),
};
```

**✅ Also right (factory form — full custom control):**
```typescript
import { flarelog, workerFetch } from "@flarelog/sdk";

export default {
  fetch: workerFetch(
    async () => flarelog({ apiKey: env?.FLARELOG_API_KEY, workerMode: true }),
    async (request, env, ctx) => {
      return new Response("Hello");
    },
  ),
};
```

Wait — `workerFetch` takes a logger, not a factory. For per-request logger
creation, use the zero-arg `flarelog()` and let the SDK auto-resolve. Or
create the logger inside the handler and use `logger.withRequest()` directly:

```typescript
import { flarelog } from "@flarelog/sdk";

export default {
  async fetch(request, env, ctx) {
    const logger = flarelog({ apiKey: env.FLARELOG_API_KEY, workerMode: true });
    return logger.withRequest(
      { request },
      ctx,
      async () => {
        logger.info("Request received", { url: request.url });
        return new Response("Hello");
      },
    );
  },
};
```

## Wrangler configuration

In `wrangler.jsonc` (or `wrangler.toml`):

```jsonc
{
  "name": "my-worker",
  "main": "src/index.ts",
  "compatibility_date": "2025-09-02",
  "compatibility_flags": ["nodejs_compat"]
}
```

**`nodejs_compat` is recommended** — it lets the SDK read `process.env.FLARELOG_API_KEY`
per-request inside the handler. Without it, the SDK falls back to the
`cloudflare:workers` `env` binding (which works but requires the zero-arg form).

## Setting the API key

**Option 1: Wrangler secret (recommended — encrypted)**
```bash
wrangler secret put FLARELOG_API_KEY
# paste fl_your_key when prompted
```

**Option 2: Dashboard secret**
Cloudflare dashboard → Workers & Pages → your Worker → Settings → Variables and Secrets → Add secret

**Option 3: Plaintext var (NOT recommended for secrets)**
```jsonc
// wrangler.jsonc
{
  "vars": {
    "FLARELOG_API_KEY": "fl_your_key"  // visible in dashboard, not encrypted
  }
}
```

Use Option 1 or 2 for the API key. Use Option 3 only for non-sensitive config
like `FLARELOG_ENVIRONMENT`.

## What `workerFetch` does

`workerFetch(logger, handler)` wraps your handler with:
- W3C trace context extraction (from `traceparent` header)
- A SERVER span for every request (name: `GET /api/users`, kind: SERVER)
- HTTP attributes: `http.request.method`, `url.path`, `url.full`, `http.response.status_code`
- Error capture with full stack trace on the span
- Flush via `ctx.waitUntil()` so logs/traces ship before the Worker suspends
- Bypass for `OPTIONS`/`HEAD` requests and `ignorePaths` matches

## Bypassing noise (OPTIONS, favicon, static assets)

```typescript
const logger = flarelog({
  apiKey: env.FLARELOG_API_KEY,
  ignorePaths: [
    "/favicon.ico",
    "/robots.txt",
    "/sitemap.xml",
    /^\/static\//,
    /^\/assets\//,
  ],
});

export default {
  fetch: workerFetch(logger, async (request, env, ctx) => {
    return new Response("Hello");
  }),
};
```

`ignorePaths` accepts strings (exact match), RegExps (pattern match), or
functions (custom predicate). Matching is against `new URL(request.url).pathname`.

## Cron Triggers

```typescript
export default {
  fetch: workerFetch(logger, async (request, env, ctx) => {
    return new Response("Hello");
  }),

  scheduled: async (event, env, ctx) => {
    const logger = flarelog({ apiKey: env.FLARELOG_API_KEY, workerMode: true });
    try {
      logger.info("Cron triggered", { cron: event.cron });
      // ... do work ...
    } catch (err) {
      logger.logError(err, { message: "Cron failed" });
    } finally {
      await logger.flush();
      ctx.waitUntil(logger.flush());
    }
  },
};
```

## Queue Consumers

```typescript
export default {
  fetch: workerFetch(logger, async (request, env, ctx) => {
    return new Response("Hello");
  }),

  queue: async (batch, env, ctx) => {
    const logger = flarelog({ apiKey: env.FLARELOG_API_KEY, workerMode: true });
    for (const message of batch.messages) {
      try {
        logger.info("Processing message", { id: message.id });
        // ... process ...
        message.ack();
      } catch (err) {
        logger.logError(err, { message: "Message failed", metadata: { id: message.id } });
        message.retry();
      }
    }
    await logger.flush();
  },
};
```

## Durable Objects

```typescript
export class MyDurableObject {
  private logger: FlareLog;

  constructor(state: DurableObjectState, env: Env) {
    this.logger = flarelog({
      apiKey: env.FLARELOG_API_KEY,
      workerMode: true,
      defaultSource: "durable-object",
    });
  }

  async fetch(request: Request) {
    return this.logger.withRequest(
      { request },
      { waitUntil: (p) => this.ctx.waitUntil(p) },
      async () => {
        this.logger.info("DO request", { doId: this.ctx.id.toString() });
        return new Response("OK");
      },
    );
  }
}
```

## Optional: Tail Worker for raw Cloudflare Workers

The SDK above captures errors thrown *during* request handling. It can't see
crashes that happen *before* your code runs:
- CPU timeout (50ms free / 30s paid)
- Memory exhaustion (128MB limit, Error 1027)
- Startup failure (bad import, missing binding)
- Error 1101 (uncaught exception before handler registers)

If you want coverage for those cases on a raw Cloudflare Worker, deploy the
optional Tail Worker. See `references/tail-worker.md` for setup.

If you are using a framework such as Hono, TanStack Start, or Lovable, this is
not required — the SDK middleware is enough.

## See also

- `references/tail-worker.md` — optional Tail Worker setup for raw Worker coverage
- `references/env-resolution.md` — how env vars work on Workers
- `references/hono.md` — if using Hono on Workers
