# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

A knowledge base and reference artifact collection for operating self-hosted Temporal Server clusters. It contains no source code — only Markdown documentation, Grafana dashboard JSON files, and YAML configuration examples. There is no build system, test runner, or CI pipeline.

The content is drawn from direct production experience and Temporal server source code analysis. It is designed to be used interactively with Claude for troubleshooting and operational guidance.

## Repository Structure

```
dynamic_config/       # Dynamic config reference, YAML example, troubleshooting
  README.md           # Full reference of all configurable parameters by service
  example.yaml        # Production-ready example with ~96 config values
  troubleshooting.md  # Patterns for RESOURCE_EXHAUSTED, cache, ES, task queue issues
  guides/             # Step-by-step operational guides (e.g. changing task queue partitions)

metrics/
  references/         # Per-metric reference docs for server and each SDK
  dashboards/
    server/           # Grafana JSON + readme for cluster overview and standby cluster
    sdk/              # Grafana JSON + readme for Go, Java (Micrometer/OTel), and Core SDKs
    troubleshooting/  # Grafana dashboards and guides for specific operational problems
  alerts/server/      # Alert planning documentation

tmp/                  # Working drafts and in-progress content (not yet finalized)
```

## Architecture and Content Organization

**Dynamic Config** covers the four Temporal services — `frontend`, `history`, `matching`, `worker` — plus `system` and `limit` namespaces. Config values can be scoped globally or constrained per namespace or task queue via the `constraints` field in YAML.

**Metrics dashboards** are Grafana JSON (v9.0+) built for Prometheus as the datasource. Dashboards use template variables for datasource, namespace, and percentile selection. Each dashboard JSON is accompanied by a detailed `-readme.md` explaining every panel group.

**Troubleshooting knowledge** is in two places:
- `dynamic_config/troubleshooting.md` — config-level remediation patterns
- `metrics/dashboards/troubleshooting/` — metric-based diagnosis guides, including `SDK_METRICS_REQUEST_FAILURES.md` and `GRAPH_PATTERNS.md`

**`tmp/`** holds drafts being developed into finished documents. These are works in progress; treat them as authoritative on their subject matter but not yet finalized in structure or placement.

## Key Operational Concepts to Know

- `host_health` (`HistoryHostHealthGauge`) is **not emitted by default** — it requires an external poller calling `AdminHandler.DeepHealthCheck` on each history pod. The per-pod gauge only ever emits `0` (absent), `1` (SERVING), or `2` (NOT_SERVING); value `3` (DECLINED_SERVING) is a fleet-level aggregate only. See `tmp/server-health.md` for the full runbook.
- Default DeepHealthCheck thresholds are intentionally conservative: 500ms average latency and 90% error ratio. A pod flipping to NOT_SERVING under these defaults indicates genuine degradation.
- Visibility `list*` operations have a default RPS of only 10 (`frontend.namespaceRPS.visibility`) — this is a common source of `RESOURCE_EXHAUSTED` errors.
- History cache is per-shard (`history.cacheMaxSize`, default 512) and host-level (`history.hostLevelCacheMaxSize`, default 128K).

## Working with This Repository

When asked to add or update content:
- Match the tone and style of existing documents — direct, production-grounded, no hedging
- Cross-reference related config values and metrics when relevant
- If adding a new dashboard JSON, include a corresponding `-readme.md` that explains each panel group
- New content belongs in the appropriate subdirectory; place drafts in `tmp/` until finalized
- The troubleshooting content is structured to be used as a Claude knowledge source — keep it scannable and organized around how problems present, not around how metrics are defined
