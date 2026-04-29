# Temporal Server Grafana Alerts — Planning Document

> **Status: Work in Progress** — This document is a planning artifact for review and discussion before implementation begins. Alert definitions, thresholds, and runbook structure are subject to change based on feedback.

---

## Overview

This document outlines the planned Grafana alert rules for the Temporal Server Dashboard. The goal is to provide a set of importable, generic alert rules that any self-hosted Temporal operator can use as a starting point, adjust to their environment, and extend as needed.

**The alerts themselves are secondary to the runbooks.** Each alert will be accompanied by a runbook that describes:
- What the alert detects
- Why it matters
- Which dashboard panels to look at first
- Which dynamic config values are relevant
- What actions to take

This is designed as a self-service operational guide — operators should be able to respond to any alert without prior deep knowledge of Temporal internals.

---

## Design Principles

**Generic defaults, expected to be tuned.** Thresholds are set based on Temporal's default dynamic config values. Every deployment is different — operators are expected to adjust thresholds to match their cluster size, workload, and SLO requirements.

**Every alert maps to a dashboard panel.** All alerts correspond directly to a panel in the Temporal Server Dashboard. When an alert fires, the operator knows exactly where to look.

**Two-tier severity.** Alerts are either Critical (page, immediate action required) or Warning (investigate soon). This avoids alert fatigue from too many severity levels.

**Compound conditions where appropriate.** Some alerts are only meaningful in combination — for example, shard movement is expected during a restart but unexpected without one. These are modeled as compound PromQL conditions rather than simple threshold breaches.

**Namespace-scoped and cluster-wide variants.** Where meaningful, alerts will have both a cluster-wide version and a per-namespace version so operators can alert on specific namespaces separately.

---

## Deliverables

| Deliverable | Description | Status |
|---|---|---|
| `temporal-server-alerts.json` | Importable Grafana alert rules | 🔲 Not started |
| `runbooks/` | One markdown runbook per alert group | 🔲 Not started |
| `README.md` | Index of all alerts with links to runbooks | 🔲 Not started |

---

## Alert Inventory

### Severity Legend
- 🔴 **Critical** — Page immediately, immediate action required
- ⚠️ **Warning** — Investigate soon, action required but not urgent

---

### Section 1 — Cluster Throughput

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 1 | Total RPS Drops to Zero | 🔴 Critical | Total RPS | Total frontend RPS drops to zero — frontend service is down, network connectivity is broken, or load balancer is misconfigured |
| 2 | Total RPS Approaching Limit | ⚠️ Warning | Total RPS | Total frontend RPS exceeds orange threshold (default 2,000 — assumes ~3 frontend instances, adjust to `frontend.rps` × instance count) |
| 3 | Total RPS Over Limit | 🔴 Critical | Total RPS | Total frontend RPS exceeds red threshold (default 7,000) — active throttling is happening, users are receiving `RpsLimit` errors |
| 4 | Namespace RPS Drops to Zero | 🔴 Critical | RPS per Namespace | Per-namespace RPS drops to zero — namespace is receiving no traffic, workers may have disconnected or namespace is misconfigured |
| 5 | Namespace RPS Approaching Limit | ⚠️ Warning | RPS per Namespace | Per-namespace RPS exceeds `frontend.namespaceRPS` warning level (default 2,400/host) |
| 6 | Namespace RPS Over Limit | 🔴 Critical | RPS per Namespace | Per-namespace RPS exceeds `frontend.namespaceRPS` hard limit — users are receiving `RpsLimit` throttle errors for this namespace |

---

### Section 2 — Shard and Workflow Lock Latencies

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 7 | Shard Lock Latency High | ⚠️ Warning | Shard Lock Latency | p99 shard lock latency exceeds 150ms |
| 8 | Shard Lock Latency Critical | 🔴 Critical | Shard Lock Latency | p99 shard lock latency exceeds 300ms |
| 9 | Workflow Lock Latency High | ⚠️ Warning | Workflow Lock Latency | p99 workflow lock latency exceeds 200ms |
| 10 | Workflow Lock Latency Critical | 🔴 Critical | Workflow Lock Latency | p99 workflow lock latency exceeds 400ms |

---

### Section 3 — Persistence

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 11 | Persistence Latency High | ⚠️ Warning | Persistence Latencies | p99 persistence latency exceeds 300ms for any operation |
| 12 | Persistence Latency Critical | 🔴 Critical | Persistence Latencies | p99 persistence latency exceeds 1s for any operation |
| 13 | Persistence QPS Approaching Limit | ⚠️ Warning | Persistence Requests Total | Total persistence req/s exceeds 7,000 (based on `history.persistenceMaxQPS` default 9,000/host) |
| 14 | Persistence Availability Degraded | ⚠️ Warning | Persistence Availability | Persistence success rate drops below 99% |
| 15 | Persistence Availability Critical | 🔴 Critical | Persistence Availability | Persistence success rate drops below 95% |
| 16 | Persistence Errors Elevated | ⚠️ Warning | Persistence Errors by Namespace | Sustained persistence error rate above zero for any namespace |
| 17 | SQL Connection Pool Near Saturation | ⚠️ Warning | SQL DB Connection Pool | Open connections exceeding 90% of configured max (SQL backends only) |
| 18 | SQL Connection Pool Saturated | 🔴 Critical | SQL DB Connection Pool | Open connections at or exceeding configured max (SQL backends only) |

---

### Section 4 — Service Latencies

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 19 | Frontend Service Latency High | ⚠️ Warning | Frontend Service Latency | p99 frontend latency exceeds 300ms (excludes PollWorkflowTaskQueue, PollActivityTaskQueue) |
| 20 | Frontend Service Latency Critical | 🔴 Critical | Frontend Service Latency | p99 frontend latency exceeds 2s |
| 21 | History Service Latency High | ⚠️ Warning | History Service Latency | p99 history latency exceeds 400ms |
| 22 | History Service Latency Critical | 🔴 Critical | History Service Latency | p99 history latency exceeds 2s |
| 23 | Matching Service Latency High | ⚠️ Warning | Matching Service Latency | p99 matching latency exceeds 400ms (excludes PollWorkflowTaskQueue, PollActivityTaskQueue, MatchingClientGetTaskQueueUserData) |
| 24 | Matching Service Latency Critical | 🔴 Critical | Matching Service Latency | p99 matching latency exceeds 2s |

---

### Section 5 — Service Requests and Errors

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 25 | Service Panic Detected | 🔴 Critical | Service Panics | Any service panic rate above zero — a Go panic means an unrecoverable error that can cause shard loss or task corruption |
| 26 | Service Error Rate Elevated | ⚠️ Warning | Service Errors by Namespace | Sustained service error rate for any namespace |
| 27 | Service Error Rate Critical | 🔴 Critical | Service Errors by Namespace | High sustained service error rate for any namespace |
| 28 | Namespace RPS Approaching Limit | ⚠️ Warning | Service Requests by Namespace | Per-namespace request rate approaching `frontend.namespaceRPS` limit |

---

### Section 6 — Throttling and Limits

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 29 | RPS/QPS Throttling Active | ⚠️ Warning | Resource Exhausted with Cause | Resource exhausted errors with cause `RpsLimit` or `QpsLimit` |
| 30 | System Overload Throttling | 🔴 Critical | Resource Exhausted with Cause | Resource exhausted errors with cause `SystemOverload` — indicates DB is overwhelmed |

---

### Section 7 — Busy Workflow Throttling

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 31 | Busy Workflow Throttling Elevated | ⚠️ Warning | Transfer Active Task Errors Workflow Busy | Sustained busy workflow error rate |
| 32 | Busy Workflow Throttling Critical | 🔴 Critical | Transfer Active Task Errors Workflow Busy | High sustained busy workflow error rate |
| 33 | Transfer Tasks Being Discarded | 🔴 Critical | Transfer Active Task Errors Discarded | Any tasks being discarded — cluster gave up retrying, tasks are lost |

---

### Section 8 — Shard Movement

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 34 | Unexpected Shard Movement | 🔴 Critical | Shards Created/Removed/Closed + Service Restarts | Shard churn rate is elevated AND service restarts == 0 in the same window — shard movement without a restart indicates DB pressure, history host crash, or membership instability |

> **Note:** Shard movement during a planned restart or scaling event is expected and not alertable. This alert uses a compound condition to filter out the expected case.

---

### Section 9 — History Timer Task Info

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 35 | Timer Task Processing Latency High | ⚠️ Warning | Timer Task Processing Latency | p99 timer processing latency exceeds 300ms |
| 36 | Timer Task Processing Latency Critical | 🔴 Critical | Timer Task Processing Latency | p99 timer processing latency exceeds 2s |
| 37 | Timer Task Scheduling Lag High | ⚠️ Warning | Timer Task Scheduling Latency | Timers firing more than 5s late |
| 38 | Timer Task Scheduling Lag Critical | 🔴 Critical | Timer Task Scheduling Latency | Timers firing more than 30s late |
| 39 | Timer Task Errors Elevated | ⚠️ Warning | Total Timer Tasks Errors | Sustained timer task error rate |

---

### Section 10 — Workflow Stats

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 40 | Workflow Execution Limit Exceeded | ⚠️ Warning | Workflow Limit Exceeded | Any workflow tasks failing due to pending activity, child workflow, or cancel request limits |
| 41 | Blob Size Limit Exceeded | ⚠️ Warning | Blob Size Errors | Any requests failing due to blob size limits — indicates SDK payloads too large |

---

### Section 11 — Workflow Execution History Info

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 42 | Workflow History Size Warning | ⚠️ Warning | Workflow History Size | p99 history size exceeds 4MB for any namespace |
| 43 | Workflow History Size Critical | 🔴 Critical | Workflow History Size | p99 history size exceeds 30MB for any namespace |
| 44 | Workflow History Event Count Warning | ⚠️ Warning | Workflow History Event Count | p99 event count exceeds 4,096 for any namespace |
| 45 | Workflow History Event Count Critical | 🔴 Critical | Workflow History Event Count | p99 event count exceeds 30,720 for any namespace |
| 46 | Mutable State Size Warning | ⚠️ Warning | Mutable State Size | p99 mutable state size exceeds 2MB for any namespace |
| 47 | Mutable State Size Critical | 🔴 Critical | Mutable State Size | p99 mutable state size exceeds 10MB for any namespace |

---

### Section 12 — SDK Workers

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 48 | Schedule to Start Latency High | ⚠️ Warning | Schedule to Start Latencies | p99 schedule-to-start latency exceeds 200ms — primary signal for insufficient workers |
| 49 | Schedule to Start Latency Critical | 🔴 Critical | Schedule to Start Latencies | p99 schedule-to-start latency exceeds 1s |
| 50 | Task Backlog Growing | ⚠️ Warning | Approximate Task Backlog | Task backlog exceeds 1,000 tasks for any namespace + task type |
| 51 | Task Backlog Critical | 🔴 Critical | Approximate Task Backlog | Task backlog exceeds 2,000 tasks |
| 52 | Tasks Persisted Rate High | ⚠️ Warning | Tasks Persisted to DB | CreateTasks rate exceeds 1,000 req/s — sustained increase indicates workers not keeping up |
| 53 | Activity ScheduleToStart Timeout | ⚠️ Warning | Activity ScheduleToStart Timeout | Any sustained activity schedule-to-start timeouts — workers not polling fast enough |
| 54 | Activity StartToClose Timeout | ⚠️ Warning | Activity StartToClose Timeout | Any sustained activity start-to-close timeouts |
| 55 | Activity Heartbeat Timeout | ⚠️ Warning | Activity Heartbeat Timeout | Any sustained heartbeat timeouts — workers crashing during long activities |
| 56 | Workflow Task Timeout (Sticky) | ⚠️ Warning | Workflow Task StartToClose Timeouts | Any sustained workflow task timeouts on sticky task queues |

---

### Section 13 — Pollers

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 57 | All Pollers Disconnected | 🔴 Critical | Total Concurrent Pollers | Concurrent pollers drops to zero for a namespace — all workers disconnected, no tasks will be processed |
| 58 | Pollers Approaching Concurrent Limit | ⚠️ Warning | Total Concurrent Pollers | Concurrent pollers approaching `frontend.namespaceCount` limit (default 1,200/instance) — will trigger `ConcurrentLimit` throttling |

---

### Section 14 — Visibility

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 59 | Visibility Availability Degraded | ⚠️ Warning | Visibility Availability | Visibility success rate drops below 99% |
| 60 | Visibility Availability Critical | 🔴 Critical | Visibility Availability | Visibility success rate drops below 95% |
| 61 | Visibility Latency High | ⚠️ Warning | Visibility Latencies per Operation | p99 visibility latency exceeds 3s |
| 62 | Visibility Latency Critical | 🔴 Critical | Visibility Latencies per Operation | p99 visibility latency exceeds 5s |
| 63 | Visibility End-to-End Latency High | ⚠️ Warning | Visibility Task End-to-End Latencies | End-to-end visibility task latency exceeds 3s — workflow state changes taking too long to appear in search |

---

### Section 15 — Cluster Replication

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 64 | Replication Stream Stuck | 🔴 Critical | Stream Health | `replication_stream_stuck` is non-zero sustained — replication has completely stalled |
| 65 | Replication Stream Errors | ⚠️ Warning | Stream Health | Sustained replication stream errors |
| 66 | Replication Stream Panic | 🔴 Critical | Stream Health | Any replication stream panics |
| 67 | Replication Latency High | ⚠️ Warning | Replication Latencies | p99 end-to-end replication latency exceeds 2s |
| 68 | Replication Latency Critical | 🔴 Critical | Replication Latencies | p99 end-to-end replication latency exceeds 4s |
| 69 | Replication DLQ Non-Empty | 🔴 Critical | DLQ Writes and Failures | `replication_dlq_non_empty` is non-zero — tasks have failed past all retries (Cassandra only) |
| 70 | Replication DLQ Enqueue Failures | 🔴 Critical | DLQ Writes and Failures | DLQ enqueue failures detected — failed tasks cannot even be preserved for inspection (Cassandra only) |
| 71 | Replication Tasks Failing | ⚠️ Warning | Replication Task Throughput | Sustained replication task failure rate |

---

### Section 16 — Authorization

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 72 | Authorization System Failure | 🔴 Critical | Authorization System Failures | Any auth system failures — auth plugin broken, all requests may start failing |
| 73 | Unauthorized Request Spike | ⚠️ Warning | Unauthorized Requests | Sustained spike in unauthorized requests — may indicate misconfigured permissions or credential issues |

---

## Summary

| Severity | Count |
|---|---|
| 🔴 Critical | 35 |
| ⚠️ Warning | 40 |
| **Total** | **73** |

---

## Related Resources

- [Temporal Server Dashboard README](./temporal-server-readme.md)
- [Temporal Dynamic Config Reference](../../dynamic_config/README.md)
- [Temporal Server Metrics Reference](https://docs.temporal.io/references/cluster-metrics)