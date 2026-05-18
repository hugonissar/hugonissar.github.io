---
title: "BigQuery Read-Only MCP Server"
tagline: "Open-source MCP server connecting AI agents to enterprise BigQuery data — with hard table allowlists, per-query scan ceilings, and built-in rate limiting."
repo: BigQuery-Read-Only-MCP-Server
language: Python
license: MIT
license_url: https://opensource.org/licenses/MIT
status: v0.1.0
order: 1
description: "Open-source Model Context Protocol (MCP) server for Google BigQuery. Hard table allowlists, per-query scan ceilings, built-in rate limiting. Works with Claude, ChatGPT, Cursor, Gemini. Self-hosted on Cloud Run for ~$0 idle cost."
---

## What this is

A production-ready **Model Context Protocol (MCP) server for Google BigQuery**
that you deploy to your own GCP project. AI agents — Claude, ChatGPT, Cursor,
Gemini, or anything else that speaks MCP — connect over HTTPS and can do exactly
two things: read the schemas of tables you explicitly allowlist, and run `SELECT`
queries against them, subject to a configurable scan budget, a result-row cap,
and a token-bucket rate limit. Nothing else.

## Why pick this over Google's official BigQuery MCP server

Google's official server is excellent if you trust the agent absolutely. This
one is for everywhere else — customer-facing chatbots, multi-tenant deployments,
agents handling untrusted input. Three differences matter most:

| | This server | Google official |
|---|---|---|
| Table access control | Hard allowlist of `(dataset, table)` pairs, enforced in the SQL parser before a job runs | IAM only — any reachable table |
| Per-query scan cap | Yes — `MAX_SCAN_MB`, enforced via dry-run | No (relies on IAM / BQ quotas) |
| Rate limiting | Built-in token bucket + burst | None (per Google's docs) |

A full feature-by-feature comparison is in the
[repository README](https://github.com/{{ site.author.github }}/BigQuery-Read-Only-MCP-Server#%EF%B8%8F-comparison-table).

## Architecture

```
┌─────────────┐    HTTPS + x-api-key    ┌────────────────────┐    IAM   ┌──────────┐
│ MCP client  │ ──────────────────────▶ │  Cloud Run service │ ───────▶ │ BigQuery │
│ (Claude,    │                         │  • SQL allowlist   │          │ tables   │
│  Cursor,    │                         │  • Dry-run cap     │          │ + views  │
│  ChatGPT)   │                         │  • Rate limiter    │          └──────────┘
└─────────────┘                         └────────────────────┘
```

Scales to zero on Cloud Run, so it costs roughly nothing at idle.

## Get it

The full source, deployment guide, and security model are on GitHub. License is MIT.

[**View on GitHub →**](https://github.com/{{ site.author.github }}/BigQuery-Read-Only-MCP-Server)
