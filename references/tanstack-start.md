# TanStack Start + Lovable Setup

## When to use this reference

- User mentions Lovable, `lovable.app`, or `lovable.dev`
- Project has `@tanstack/react-start` in `package.json`
- Project has `src/start.ts` or `src/start.tsx`
- Project uses `createStart()` or `createMiddleware()` from `@tanstack/react-start`

## Compatibility

Requires `@tanstack/react-start` >= 1.0.0 (stable). Verified against v1.168.x.

**Do NOT use `getRequestEvent()`** — it was removed before v1 stable and is not
exported by `@tanstack/react-start` >= 1.0.0. The zero-arg middleware uses the
`cloudflare:workers` `env` binding instead.

## Install

```bash
npm install @flarelog/sdk
```

`@tanstack/react-start` is a peer dependency — Lovable projects already have it.
For non-Lovable projects, install it if missing:

```bash
npm install @tanstack/react-start
```

## Setup

### Zero-config (recommended — works on Lovable, Cloudflare Workers, Vercel, Node)

In `src/start.ts`:

```typescript
import { createStart } from "@tanstack/react-start";
import { tanstackStartMiddleware } from "@flarelog/sdk/tanstack-start";

export const startInstance = createStart(() => ({
  requestMiddleware: [tanstackStartMiddleware() as never],
}));
```

The `as never` cast is needed because TanStack Start's middleware type inference
doesn't automatically accept the FlareLog builder. If the user's TS setup infers
the builder type directly, they can omit it.

**Do NOT pass a logger instance** when using the zero-arg form. The middleware
auto-creates a logger that reads `FLARELOG_API_KEY` from:
1. `process.env` (Node, Vercel, Workers with `nodejs_compat`)
2. The `cloudflare:workers` `env` binding (Workers without `nodejs_compat`, Lovable)

See `references/env-resolution.md` for the full resolution matrix.

### Eager logger (custom config — Node/Vercel only)

On Node or Vercel, `process.env` is available at module load, so this is safe:

```typescript
import { createStart } from "@tanstack/react-start";
import { flarelog } from "@flarelog/sdk";
import { tanstackStartMiddleware } from "@flarelog/sdk/tanstack-start";

const logger = flarelog({
  apiKey: process.env.FLARELOG_API_KEY!,
  sampleRate: 0.1,
});

export const startInstance = createStart(() => ({
  requestMiddleware: [tanstackStartMiddleware(logger) as never],
}));
```

**Do NOT use this form on Cloudflare Workers / Lovable** — `process.env.FLARELOG_API_KEY`
is `undefined` at module load. Use the zero-arg form instead.

### Factory (full custom control — multi-tenant, custom env source)

```typescript
import { createStart } from "@tanstack/react-start";
import { flarelog } from "@flarelog/sdk";
import { tanstackStartMiddleware } from "@flarelog/sdk/tanstack-start";
import { env } from "cloudflare:workers";

export const startInstance = createStart(() => ({
  requestMiddleware: [
    tanstackStartMiddleware(() => {
      return flarelog({
        apiKey: env.FLARELOG_API_KEY,
        workerMode: true,
        sampleRate: 0.1,
      });
    }) as never,
  ],
}));
```

Note: `import { env } from "cloudflare:workers"` only works on the Cloudflare
Workers runtime. On Node/Vercel it throws — guard with try/catch or use
`process.env` directly there.

## Using the logger in routes and server functions

The middleware attaches `context.logger` (a child logger with traceId, method,
path) and `context.traceId` to every request. Access them in handlers:

```typescript
import { createServerFn } from "@tanstack/react-start";

export const getUser = createServerFn({ method: "GET" })
  .handler(async ({ context }) => {
    context.logger.info("getUser called", { userId: 42 });
    return { id: 42, name: "Ada" };
  });
```

Or in route loaders:

```typescript
import { createFileRoute } from "@tanstack/react-router";

export const Route = createFileRoute("/api/users/$id")({
  server: {
    middleware: [tanstackStartMiddleware() as never],
    handlers: {
      GET: async ({ context, params }) => {
        context.logger.info("Fetching user", { userId: params.id });
        return Response.json({ id: params.id });
      },
    },
  },
});
```

## Per-route vs global middleware

The global setup in `src/start.ts` covers every request. For per-route or
per-server-function middleware, pass the middleware directly:

```typescript
// Per server function
export const getUser = createServerFn({ method: "GET" })
  .middleware([tanstackStartMiddleware() as never])
  .handler(async ({ context }) => {
    context.logger.info("getUser called");
    return { id: 1 };
  });

// Per route
export const Route = createFileRoute("/api/orders/$id")({
  server: {
    middleware: [tanstackStartMiddleware() as never],
    handlers: { /* ... */ },
  },
});
```

## Lovable-specific instructions

If the user is on Lovable, they edit via chat — they don't write code directly.
Tell them to paste this into the Lovable chat:

```
Add FlareLog logging to my app.

1. Install @flarelog/sdk
2. In src/start.ts, add the zero-config middleware:
   import { tanstackStartMiddleware } from "@flarelog/sdk/tanstack-start";
   Add tanstackStartMiddleware() to the requestMiddleware array in createStart().
3. I'll add FLARELOG_API_KEY as a secret in Lovable's Secrets panel.
```

Then tell them to add the API key in Lovable's project settings under
**Secrets / Environment variables**:
- Name: `FLARELOG_API_KEY`
- Value: `fl_their_key_here`
- **Do NOT prefix with `VITE_`** — that leaks the key to the browser

## Merging with existing CSRF middleware

If `src/start.ts` already has CSRF middleware (Lovable generates this by default),
merge them:

```typescript
import { createStart, createCsrfMiddleware } from "@tanstack/react-start";
import { tanstackStartMiddleware } from "@flarelog/sdk/tanstack-start";

export const startInstance = createStart(() => ({
  requestMiddleware: [
    createCsrfMiddleware(),
    tanstackStartMiddleware() as never,
  ],
}));
```

## What the middleware does

- Extracts or generates a traceId from the `x-trace-id` header (or W3C `traceparent`)
- Creates a child logger with `{ source: "tanstack-start", traceId, method, path }`
- Attaches `context.logger` and `context.traceId` to the downstream context
- Logs "Request completed" with duration and status (4xx → WARN, 5xx → ERROR)
- Calls `await logger.flush()` after each request (critical on Workers —
  prevents the Worker from suspending before logs ship)
- Captures thrown errors with full context, then re-throws

## Status code mapping

The middleware reads the status from `result.response.status` (the v1
`RequestServerResult` shape is `{ request, pathname, context, response }`).
It also handles raw `Response` returns (short-circuit case) and legacy
`{ status }` shapes.

| Status range | Log level |
|-------------|-----------|
| 2xx-3xx | INFO |
| 4xx | WARN |
| 5xx | ERROR |
| No status available | INFO |

## Troubleshooting

### "No backend configured — falling back to console-only logging"

The SDK couldn't find `FLARELOG_API_KEY`. On Lovable/Workers, this means:
- The zero-arg middleware couldn't read the `cloudflare:workers` binding, AND
- `process.env.FLARELOG_API_KEY` is not set

Fix: add the key in Lovable's Secrets panel (without `VITE_` prefix), or
enable `nodejs_compat` in `wrangler.jsonc` so `process.env` is populated
per-request.

### "Logs work in dev but not in preview/production"

Dev runs on Node where `process.env` works at module load. Preview/production
runs on Workers where it doesn't. Use the zero-arg middleware form — it reads
the `cloudflare:workers` binding on the first request.

### TypeScript errors with `as never`

The `as never` cast is needed because TanStack Start's middleware type inference
doesn't automatically accept the FlareLog builder. If the user's TS setup infers
the builder type directly, they can omit it. This is a TanStack Start typing
quirk, not a FlareLog bug.

## See also

- `references/env-resolution.md` — full env var resolution matrix
- `references/tail-worker.md` — Tail Worker setup for crash capture on Workers
- `references/mcp-setup.md` — connect Cursor/Claude for AI debugging
