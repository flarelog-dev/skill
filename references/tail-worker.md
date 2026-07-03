# Tail Worker Setup (raw Cloudflare Workers only)

> **Heads up:** If you're using Lovable, TanStack Start, Next.js on Cloudflare,
> or any other framework, you **do not** need a Tail Worker. Use the SDK
> middleware for your framework instead (see `references/tanstack-start.md`).
> This guide is only for plain/raw Cloudflare Workers users who want extra
> out-of-band crash coverage.

## What the Tail Worker does

The Tail Worker runs **out-of-band**, after your main Worker finishes — whether
it succeeded or crashed. It receives the execution outcome from Cloudflare's
runtime, including:

- The uncaught exception with full stack trace
- CPU time used (so you can see if you hit the limit)
- Memory usage
- Request URL, method, and headers
- Duration and response status
- The outcome type: `ok`, `exception`, `exceededCpu`, `exceededMemory`, etc.

Even if your Worker dies on line 1, the Tail Worker captures it. Even if you
didn't write any logging code, the Tail Worker captures it. It runs outside your
app — like a security camera that watches from the outside.

## Why you need it (even if you have the SDK)

The SDK (`@flarelog/sdk`) captures errors thrown *during* request handling. But
it can't see crashes that happen *before* your code runs:

| Crash type | SDK sees it? | Tail Worker sees it? |
|-----------|:---:|:---:|
| Error in your fetch handler | ✅ | ✅ |
| CPU timeout (killed mid-execution) | ❌ | ✅ |
| Memory exhaustion (Error 1027) | ❌ | ✅ |
| Startup failure (bad import, missing binding) | ❌ | ✅ |
| Error 1101 (uncaught before handler registers) | ❌ | ✅ |
| Subrequest timeout | ❌ | ✅ |

If you're on raw Cloudflare Workers and want coverage for those crashes, the
Tail Worker is the way to get it. Without it, you'd be blind to the most
damaging failures.

## Setup (5 minutes)

### 1. Clone the template

```bash
git clone https://github.com/flarelog-dev/tail-worker.git
cd tail-worker
npm install
```

### 2. Set your FlareLog API key

```bash
wrangler secret put FLARELOG_API_KEY
# paste your fl_ key when prompted
```

### 3. (Optional) Set up alert webhook

```bash
wrangler secret put ALERT_WEBHOOK_URL
# paste your Slack/Discord webhook URL
```

### 4. Point it at your app's Worker

Edit `wrangler.toml`:

```toml
name = "flarelog-tail-worker"
main = "src/index.ts"
compatibility_date = "2025-09-02"

[[tail_consumers]]
service = "your-app-worker-name"  # ← change this to your app's Worker name

[vars]
TRACK_COST = "true"           # enable cost burn alerts
DAILY_SPEND_LIMIT = "5"       # alert at $5/day
```

### 5. Deploy

```bash
npm run deploy
```

## How to find your app's Worker name

- **Wrangler**: the `name` field in your app's `wrangler.toml` or `wrangler.jsonc`
- **Cloudflare dashboard**: Workers & Pages → your app → the name in the sidebar

The Tail Worker's `[[tail_consumers]]` `service` field must match this name
exactly.

## Cost tracking (optional but recommended)

The Tail Worker estimates the cost of every request in real-time using
Cloudflare's public pricing. Enable it in `wrangler.toml`:

```toml
[vars]
TRACK_COST = "true"
DAILY_SPEND_LIMIT = "5"        # alert at $5/day
KV_WRITE_ALERT_THRESHOLD = "100000"  # alert at 100k KV writes/day
```

When spend crosses a threshold, the Tail Worker sends a POST to your
`ALERT_WEBHOOK_URL` with a JSON payload:

```json
{
  "type": "cost_alert",
  "threshold": "5",
  "current_spend": "5.23",
  "resource": "kv_writes",
  "count": 142000,
  "timestamp": "2026-01-27T10:30:00Z"
}
```

## Slack webhook setup

1. Go to Slack → your channel → Integrations → Add apps → Incoming Webhooks
2. Create a webhook URL
3. `wrangler secret put ALERT_WEBHOOK_URL` and paste the URL

## Discord webhook setup

1. Discord → your server → channel settings → Integrations → Webhooks
2. Create a webhook and copy the URL
3. `wrangler secret put ALERT_WEBHOOK_URL` and paste the URL

## Does the Tail Worker slow down my app?

No. Tail Workers run asynchronously, after your response is sent to the user.
They have zero impact on your app's latency. Cloudflare doesn't even charge for
Tail Worker invocations on most plans.

## Does the Tail Worker work with multiple app Workers?

Yes. Add multiple `[[tail_consumers]]` entries:

```toml
[[tail_consumers]]
service = "my-app-worker"

[[tail_consumers]]
service = "my-api-worker"

[[tail_consumers]]
service = "my-auth-worker"
```

One Tail Worker can watch many app Workers. All logs ship to the same FlareLog
project (identified by your API key).

## Troubleshooting

### "My Tail Worker deployed but I don't see logs"

1. Check that `service` in `wrangler.toml` matches your app's Worker name exactly
2. Check that `FLARELOG_API_KEY` is set as a secret (not a var)
3. Hit your app a few times — Tail Worker logs appear in FlareLog within seconds
4. Check the Tail Worker's logs in Cloudflare dashboard for errors

### "I'm getting Error 1101 but the Tail Worker shows nothing"

The Tail Worker only captures requests that Cloudflare's runtime passes to it.
If your app Worker fails to deploy (syntax error, bad wrangler config), the
Tail Worker has nothing to tail. Check your app's deploy status first.

### "Cost alerts aren't firing"

1. Verify `TRACK_COST = "true"` is in `[vars]` (not secrets — vars)
2. Verify `ALERT_WEBHOOK_URL` is set as a secret
3. Check that your spend has actually crossed `DAILY_SPEND_LIMIT`
4. Check the Tail Worker logs for webhook delivery errors

## See also

- `references/cloudflare-workers.md` — SDK setup for raw Workers
- `references/tanstack-start.md` — SDK setup for Lovable/TanStack Start
- `references/env-resolution.md` — env var resolution on Workers
