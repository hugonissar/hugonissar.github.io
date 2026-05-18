---
title: "GA4 Anomaly Detector"
tagline: "Statistical anomaly detection over GA4 BigQuery exports, narrated by an LLM. The pipeline does the math in code (STL, PELT, JS-divergence); the LLM only writes the prose. Includes a CLI and an MCP server for Claude Desktop, Cursor, and Gemini CLI."
repo: GA4-Anomaly-Detector
language: Python
license: MIT
license_url: https://opensource.org/licenses/MIT
status: Pre-release
order: 4
description: "Statistical anomaly detection for GA4 BigQuery exports. Three detectors run in code (STL residual z-score, PELT change points, Jensen-Shannon divergence for mix shifts); an LLM narrates the structured findings into a markdown report. Available as a CLI or as an MCP server for Claude Desktop, Cursor, and other MCP-compatible clients. Pre-release; intended for single-user local use."
---

## What this is

A statistical anomaly detection pipeline for **GA4 BigQuery exports** with
the trust boundary drawn explicitly: **the math runs in code; the LLM only
writes the prose.** Three detectors run over your data, produce structured
findings, and pass them to a Gemini-narrated (or template-rendered)
markdown report you can drop into Slack, email, or a doc.

The motivating example is in the README: a published benchmark of
Google's official GA4 MCP server reporting a *+35% increase* when the
actual trend was a *−60% decrease*. The cause was structural — the agent
was handed raw query rows and asked to analyze them. This tool inverts
that ordering so the same failure mode can't happen.

## How it works

Three detectors, each picked because the alternative was wrong for
GA4-shaped data:

- **Point anomalies — STL residual z-score.** Decomposes each metric into
  trend, weekly seasonal, and residual, then z-scores the residual. A
  naive rolling mean would flag every Saturday as anomalous; this doesn't.
- **Change points — PELT with RBF cost.** Distinguishes a one-day spike
  (campaign launch) from a sustained shift (site migration). The RBF cost
  is roughly scale-invariant, so the same penalty works for `sessions`
  (thousands) and `conversion_rate` (single digits).
- **Mix shifts — Jensen-Shannon divergence.** Catches the case GA4
  dashboards hide: total sessions look flat, but direct doubled while
  organic collapsed. Operates on share-of-voice distributions across
  adjacent windows.

The LLM receives only the structured findings — never the daily metric
values, never the raw event rows. The system prompt forbids speculation
about causes that aren't corroborated by other findings.

## Two ways to run it

- **CLI** (`python cli.py sample` or `python cli.py run ...`). Once-per-
  invocation, exits when done. Safe to run anywhere.
- **MCP server** (`python mcp_server.py`). Exposes the analyze pipeline
  as a tool callable from Claude Desktop, Cursor, Claude Code, or any
  MCP-compatible client. Intended for **single-user local use only** —
  see "Status and caution" below.

The full CLI reference, including every flag, every environment variable,
and the exit-code semantics, is in the companion post:
[*When +35% is really −60%: a GA4 anomaly detector where the LLM only
writes the report*](/blog/ga4-anomaly-detector/).

## Status and caution

This pipeline is **pre-release**. The CLI is the well-tested path. The
MCP server intentionally ships without the guardrails my other
open-source MCP server provides — there's no table allowlist, no
per-query scan ceiling, no rate limiter, no API key auth. That's fine for
local single-user use (`python mcp_server.py` called by your own Claude
Desktop, against your own GA4 export), but it is **not** suitable for
multi-tenant or customer-facing deployment.

If you want an MCP server with hard guardrails for production use, the
right project for that is
[**BigQuery Read-Only MCP Server**](/projects/bigquery-readonly-mcp-server/) —
allowlists, scan caps, rate limiting, API key auth, designed to be safely
exposed over HTTPS. The two projects are complementary.

## Get it

Full source — detectors, BigQuery fetcher, narrative renderer, CLI, and
MCP server:

[**View on GitHub →**](https://github.com/{{ site.author.github }}/GA4-Anomaly-Detector)
