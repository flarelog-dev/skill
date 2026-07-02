# MCP Server Setup (Cursor + Claude Desktop)

## What the MCP server does

The FlareLog MCP server (`mcp.flarelog.dev`) exposes your production logs,
errors, traces, and cost data to AI tools like Cursor and Claude Desktop. Once
connected, your AI can:

- Query your logs in natural language ("What broke in the last hour?")
- Search for specific errors by traceId, URL, or message
- See the full stack trace of a crash
- Correlate errors with recent deploys
- Suggest fixes based on your actual production data (Cursor can apply them directly)

This is the killer feature for vibe coders — your AI can debug production
without you copy-pasting error messages.

## Cursor setup

Create `.cursor/mcp.json` in your project root:

```json
{
  "mcpServers": {
    "flarelog": {
      "url": "https://mcp.flarelog.dev",
      "headers": {
        "Authorization": "Bearer fl_your_api_key_here"
      }
    }
  }
}
```

Replace `fl_your_api_key_here` with your FlareLog API key (from
[flarelog.dev/login](https://flarelog.dev/login)). Restart Cursor.

## Claude Desktop setup

Edit `claude_desktop_config.json`:
- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "flarelog": {
      "url": "https://mcp.flarelog.dev",
      "headers": {
        "Authorization": "Bearer fl_your_api_key_here"
      }
    }
  }
}
```

Restart Claude Desktop.

## What you can ask

Once connected, ask your AI in natural language:

- "What errors happened in production in the last hour?"
- "Show me the stack trace for the crash at 3:42 AM"
- "Which API endpoint is throwing the most 500s?"
- "Find the request that caused this traceId and suggest a fix"
- "Are there any patterns in the errors from the last deploy?"
- "What's my Cloudflare bill looking like today?"

## Is it safe?

Yes:
- **Read-only**: the MCP server only exposes READ operations — your AI can't
  modify or delete logs
- **Scoped API key**: the key is per-project, so the AI only sees logs for that
  project
- **No data leaves your dashboard**: the MCP server queries your FlareLog
  dashboard; logs stay in FlareLog
- **Revoke anytime**: rotate your API key to instantly revoke AI access

## Which tools support MCP?

| Tool | MCP support | Config file |
|------|:-----------:|-------------|
| Cursor | ✅ | `.cursor/mcp.json` |
| Claude Desktop | ✅ | `claude_desktop_config.json` |
| Windsurf | ✅ | Check their MCP docs |
| Zed | ✅ | `~/.config/zed/settings.json` |
| ChatGPT | ❌ | Not yet — use the FlareLog dashboard directly |
| GitHub Copilot | ❌ | Not yet |

## See also

- `references/tail-worker.md` — captures the crashes the MCP server will show your AI
