# Express Setup

## When to use this reference

- User has `express()` or `app.use`
- App uses `(req, res, next)` pattern
- Common on Node.js servers

## Install

```bash
npm install @flarelog/sdk express
```

## Setup

```typescript
import express from "express";
import { flarelog } from "@flarelog/sdk";
import { expressMiddleware, expressErrorHandler } from "@flarelog/sdk/express";

const logger = flarelog({ apiKey: process.env.FLARELOG_API_KEY! });

const app = express();

// Request logging middleware (register first)
app.use(expressMiddleware(logger));

// Error handler (register last, after all routes)
app.use(expressErrorHandler(logger));

app.get("/api/hello", (req, res) => {
  req.logger.info("Hello!"); // req.logger is attached by the middleware
  res.json({ message: "Hello from Express!" });
});

app.listen(3000);
```

## What the middleware does

`expressMiddleware(logger)`:
- Extracts traceId from `x-trace-id` header or `crypto.randomUUID()`
- Creates a child logger with `{ source: "express", traceId, method, path, ip }`
- Attaches `req.logger` and `req.traceId`
- Listens on `res.on("finish")` to log completion with duration and status
- Maps status to log level: 5xx → ERROR, 4xx → WARN, else INFO

`expressErrorHandler(logger)`:
- Catches errors passed to `next(err)`
- Logs via `req.logger.logError(err, { message: "Express error" })`
- Sends a 500 response if headers haven't been sent

## Environment variables

Express runs on Node.js — `process.env` works at module load:

```bash
# .env
FLARELOG_API_KEY=fl_your_key
FLARELOG_ENVIRONMENT=production
```

## See also

- `references/env-resolution.md` — env var matrix
