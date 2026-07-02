# Next.js Setup

## When to use this reference

- User has `next.config.js` / `next.config.ts`
- App uses `pages/api/` (Pages Router) or `app/api/` (App Router)
- Uses `NextApiRequest` / `NextApiResponse` or Route Handlers

## Install

```bash
npm install @flarelog/sdk
```

## Setup

### Pages Router — API routes (`pages/api/*.ts`)

```typescript
// pages/api/hello.ts
import { flarelog } from "@flarelog/sdk";
import { withFlareLog } from "@flarelog/sdk/next";

const logger = flarelog({ apiKey: process.env.FLARELOG_API_KEY! });

export default withFlareLog(logger, async (req, res) => {
  req.logger.info("Processing request");
  res.json({ message: "Hello from Next.js!" });
});
```

The wrapper attaches `req.logger` (child logger with traceId, method, path) and
`req.traceId` to every request.

### App Router — Route Handlers (`app/api/[name]/route.ts`)

```typescript
// app/api/hello/route.ts
import { flarelog } from "@flarelog/sdk";
import { withNextRouteHandler } from "@flarelog/sdk/next";

const logger = flarelog({ apiKey: process.env.FLARELOG_API_KEY! });

export const GET = withNextRouteHandler(logger, async (request) => {
  return Response.json({ message: "Hello from App Router!" });
});
```

### Edge Middleware (`middleware.ts`)

```typescript
// middleware.ts
import { flarelog } from "@flarelog/sdk";
import { withNextMiddleware } from "@flarelog/sdk/next";

const logger = flarelog({ apiKey: process.env.FLARELOG_API_KEY! });

export default withNextMiddleware(logger, async (request) => {
  // Your middleware logic
  return NextResponse.next();
});
```

## Environment variables

Next.js on Vercel or Node.js — `process.env` works at module load:

```bash
# .env.local
FLARELOG_API_KEY=fl_your_key
FLARELOG_ENVIRONMENT=production
FLARELOG_RELEASE=1.2.3
```

For browser-side logging, create a separate client logger with a public key:
```bash
NEXT_PUBLIC_FLARELOG_CLIENT_KEY=fl_public_key
```

## What the wrappers do

- `withFlareLog` (Pages Router): wraps `(req, res) => ...`, uses `res.on("finish")`
  to capture the final status code, logs completion with duration
- `withNextRouteHandler` (App Router): wraps `(request) => Response`, extracts
  status from the Response object, logs completion
- `withNextMiddleware` (Edge): wraps `(request) => Response`, logs middleware
  execution

All three:
- Extract traceId from `x-trace-id` or W3C `traceparent` header
- Create a child logger with `{ source: "nextjs", traceId, method, path }`
- Capture thrown errors, log them, then re-throw
- Call `logger.flush()` after the handler completes

## See also

- `references/vercel.md` — if deploying standalone (not Next.js) to Vercel
- `references/env-resolution.md` — env var matrix
