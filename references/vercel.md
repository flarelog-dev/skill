# Vercel Setup (Standalone — not Next.js)

## When to use this reference

- User has `vercel.json` or `api/` directory
- Uses Vercel Serverless Functions or Edge Functions
- NOT using Next.js (if using Next.js, see `references/nextjs.md`)

## Install

```bash
npm install @flarelog/sdk
```

## Setup

### Serverless Functions (Node.js runtime) — `api/*.ts`

```typescript
// api/hello.ts
import { flarelog } from "@flarelog/sdk";
import { withVercelServerless } from "@flarelog/sdk/vercel";

const logger = flarelog({ apiKey: process.env.FLARELOG_API_KEY! });

export default withVercelServerless(logger, async (req, res) => {
  req.logger.info("Processing request");
  res.json({ message: "Hello from Vercel!" });
});
```

### Edge Functions — `api/edge.ts`

```typescript
// api/edge.ts
import { flarelog } from "@flarelog/sdk";
import { withVercelEdge } from "@flarelog/sdk/vercel";

export const config = { runtime: "edge" };

const logger = flarelog({ apiKey: process.env.FLARELOG_API_KEY! });

export default withVercelEdge(logger, async (request) => {
  return new Response("Hello from the edge!");
});
```

## Environment variables

Set in Vercel dashboard → Project → Settings → Environment Variables:

```
FLARELOG_API_KEY=fl_your_key
FLARELOG_ENVIRONMENT=production
FLARELOG_RELEASE=<vercel-git-commit-sha>
```

`process.env` works at module load on Vercel (both Serverless and Edge) —
safe to create the logger eagerly.

## What the wrappers do

`withVercelServerless`:
- Extracts traceId from `x-trace-id` or `traceparent` header
- Attaches `req.logger` and `req.traceId`
- Uses `res.on("finish")` to capture final status
- Logs completion with duration and status
- Captures errors, sends 500 if headers not sent, re-throws

`withVercelEdge`:
- Delegates to `logger.withRequest()` for full OTel span treatment
- Extracts W3C `traceparent`, creates a SERVER span
- Flushes telemetry before the function invocation ends

## detectVercelEnv helper

```typescript
import { detectVercelEnv } from "@flarelog/sdk/vercel";

const venv = detectVercelEnv();
if (venv) {
  console.log(`Running on Vercel ${venv.environment} in ${venv.region}`);
  // venv.commitSha, venv.commitRef, venv.projectId, venv.deploymentId
}
```

## See also

- `references/nextjs.md` — if using Next.js on Vercel
- `references/env-resolution.md` — env var matrix
