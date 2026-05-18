---
title: "Connecting Claude Desktop to BigQuery via MCP in 5 minutes"
date: 2026-05-16
last_modified_at: 2026-05-16
repo: BigQuery-Read-Only-MCP-Server
tags: [mcp, claude, bigquery, tutorial]
description: >-
  A five-minute walkthrough of connecting Claude Desktop to Google BigQuery
  through a self-hosted MCP server. Includes the full claude_desktop_config.json,
  the `mcp-proxy` invocation, and how to verify the link from inside Claude.
excerpt: >-
  Once a BigQuery MCP server is deployed on Cloud Run, hooking Claude Desktop
  up to it is a single JSON config edit. Here's the exact config, the
  `mcp-proxy` flags that matter, and how to confirm Claude can actually see
  your tables.
---

Once a BigQuery MCP server is deployed and reachable on HTTPS, the
**Claude Desktop** side of the connection is a single JSON config edit.
This post walks through the exact config, the `mcp-proxy` flags that matter,
and a one-line query to verify the link works end-to-end.

If you don't have a BigQuery MCP server running yet, the prerequisite is
deploying one. The self-hosted alternative I maintain takes about ten minutes
to deploy on Cloud Run — guide is in the
[repository README](https://github.com/{{ site.author.github }}/BigQuery-Read-Only-MCP-Server).

## What you need before starting

- Claude Desktop installed (macOS, Windows, or Linux).
- A reachable MCP endpoint — i.e. an HTTPS URL ending in `/mcp` that responds
  to streamable-HTTP MCP requests. For the self-hosted server above, the URL
  is whatever Cloud Run assigned, ending in `.a.run.app/mcp`.
- The MCP API key you set in `MCP_API_KEY` at deploy time.
- `uv` (or `uvx`) installed — Astral's Python launcher. On macOS:
  `brew install uv`. On Linux: `curl -LsSf https://astral.sh/uv/install.sh | sh`.

## Step 1: locate your Claude Desktop config

The config lives at:

| OS | Path |
|---|---|
| macOS | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| Windows | `%APPDATA%\Claude\claude_desktop_config.json` |
| Linux | `~/.config/Claude/claude_desktop_config.json` |

Open it. If it doesn't exist yet, create it with `{}` as the only contents.

## Step 2: add the MCP server entry

Replace `<your-cloud-run-url>` with your service URL and `<your-api-key>`
with the value from `MCP_API_KEY`:

```json
{
  "mcpServers": {
    "bigquery": {
      "command": "uvx",
      "args": [
        "mcp-proxy",
        "--transport", "streamablehttp",
        "-H", "x-api-key", "<your-api-key>",
        "https://<your-cloud-run-url>/mcp"
      ]
    }
  }
}
```

A few notes on what each piece does:

- `"command": "uvx"` runs `mcp-proxy` in an isolated Python environment
  managed by `uv`. If `uvx` isn't on your `PATH` from Claude's perspective,
  use the absolute path instead — e.g. `/home/you/.local/bin/uvx` on Linux
  or `/Users/you/.local/bin/uvx` on macOS.
- `--transport streamablehttp` selects the right MCP transport. The
  self-hosted server above speaks streamable HTTP; Cloud Run plays well with
  it because connections terminate fast.
- `-H "x-api-key" "<your-api-key>"` injects the API key as an HTTP header.
  The MCP server validates it in constant time before any work happens —
  bad keys get a `401` immediately.

## Step 3: restart Claude Desktop

Quit and relaunch. The first start after a config change does an MCP
handshake against every server in the file. If the handshake fails, Claude
shows a red badge on the MCP icon and the server is silently disabled.

## Step 4: verify from inside Claude

Open a new chat and ask:

> *"What tables do you have access to via the bigquery MCP server?"*

Claude should call `get_table_schema` and return your allowlisted tables.
If the server's allowlist is `analytics.events, reporting.daily`, the answer
will name exactly those two.

If you instead get *"I don't have access to a bigquery server"* or
*"The bigquery tool is unavailable"*:

- Check the MCP icon at the bottom of the Claude composer. A red dot
  means the handshake failed.
- Tail Cloud Run logs and look for a request from your client IP. If there's
  no request at all, it's a client-side problem (config typo, wrong URL,
  `uvx` not on PATH). If you see a `401`, the API key in the config doesn't
  match the secret.
- The most common cause of "connects but no tools" is a hostname mismatch
  between what you typed in the config and what your Cloud Run service
  actually answers on. Copy the URL straight from the gcloud deploy output.

## Step 5: ask a real question

Once verified, try a question that actually exercises the data:

> *"What were the top five referring sources to my site in the last 7 days,
> by sessions?"*

Claude will compose a `SELECT` against the events table, the MCP server will
dry-run it (rejecting it if it exceeds `MAX_SCAN_MB`), execute it, return
truncated results, and Claude will summarize them. The whole roundtrip is
usually 2–4 seconds.

## What to do next

A few things worth knowing once it's working:

- **The scan ceiling is the single most important config.** Default is 100 MB.
  For exploratory work on a small GA4 export, leave it. For a production
  setup against billions of rows, drop it to 25 MB to keep Claude's
  free-text queries cheap.
- **The token bucket throttles bad behavior, not legitimate use.** Default
  is 20 queries per minute with a burst of 5. If Claude trips the rate limit
  the user sees a clear "rate limit exceeded" error, not a silent failure.
- **The schema cache TTL is per-instance.** If you change a table's schema
  in BigQuery, the server's local cache (default 5 minutes) keeps the old
  version. Either wait for the TTL or hit the `/admin/invalidate-cache`
  endpoint with the admin key.
- **Don't expose this to multiple users on one API key.** Per-user audit
  trail isn't supported in v0.1. If you need it, put an identity-aware proxy
  in front, or fork and add OAuth.

## FAQ

### Does this work with Cursor, Windsurf, ChatGPT, or other MCP clients?

Yes. The MCP server is client-agnostic — anything that speaks streamable-HTTP
MCP works. Cursor uses the same `mcpServers` JSON config in its settings.
Claude Code uses the `claude mcp add` CLI command. ChatGPT supports MCP via
the Responses API; the configuration UI is slightly different but the
endpoint URL and API key are the same.

### Can I share one MCP server across my team?

You can, but everyone shares the same API key and the audit log shows the
service account, not individual users. For shared use, deploy one server
behind Cloud IAP and let each person authenticate with their own Google
identity. The MCP API key + IAP combo gives you per-user attribution in
Cloud Audit Logs.

### Do I need to keep `mcp-proxy` running separately?

No. Claude Desktop spawns and supervises the `uvx mcp-proxy` subprocess for
the lifetime of the desktop app. When you quit Claude, the proxy exits.
There's nothing to manage between sessions.

### How do I revoke access?

Rotate the API key in Secret Manager (`gcloud secrets versions add
mcp-api-key --data-file=-`) and update every client config. The old version
keeps working until you disable it explicitly.

## Source code

Server source — one Python file, MIT-licensed:
[github.com/{{ site.author.github }}/BigQuery-Read-Only-MCP-Server](https://github.com/{{ site.author.github }}/BigQuery-Read-Only-MCP-Server).
