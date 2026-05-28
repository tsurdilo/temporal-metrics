# Changelog — Temporal History Host Health Dashboard

## v1.3.0 — 2026-05-28

### Added
- **Multi-cluster filtering.** Two new template variables:
  - `cluster_label` (constant, default `clusterId`, hidden) — Prometheus label name identifying a cluster. Edit in dashboard settings if your metrics use a different label (e.g. `cluster_id`, `cluster`).
  - `cluster` (query, `label_values(${cluster_label})`, includeAll, allValue `.*`, default **All**) — filters every panel by cluster.
- All panel selectors now include `${cluster_label}=~"$cluster"`. Backward compatible: with **All** selected the expansion is `=~".*"`, which matches series even when the cluster label is absent — single-cluster setups with no `clusterId` label render identically to v1.2.0 with no configuration needed.

---

## v1.2.0 — 2026-05-12

### Changed
- Renamed **History Pods Count** panel to **Fleet Size Change** — the name better reflects that the signal covers both unexpected pod loss and planned scale-down events, which are indistinguishable at the metric level.

---

## v1.1.0 — 2026-05-12

### Added
- **Fleet Size Change** stat panel in the History Host Health row — detects pods that stopped emitting `host_health` entirely (e.g. crashed or killed), which do not appear as NOT_SERVING. Uses `max_over_time` over a 1-hour window as the fleet baseline; turns red when current pod count falls below that baseline.

---

## v1.0.0 — 2026-05-12

Initial versioned release.

### Added
- History Host Health row: `host_health` gauge panels (Pods SERVING / NOT_SERVING / DECLINED_SERVING, Total Pods Reporting, Metric Freshness, Pod Count by Health State, Pod Health State Percentage, Per-Pod Health State)
- Service Readiness (gRPC Health) row: Frontend / History / Matching Pods Ready, Ready Pod Count Over Time
- Persistence Health row: Persistence Latency, Errors by Type, Availability, SQL DB Connection Pool, Session Refresh Attempts and Failures
- History RPC Health row: History Service Latency, Errors by Type, Shard Ownership Lost, Membership Changes, Service Panics
- Shard Acquisition Health and Movement row: Shards Per Pod, Shard Acquisition Latency, Shards Created, Shards Removed, Shards Closed, Service Restarts

### Fixed
- Used correct metric name `shard_closed_count` (not `sharditem_closed_count`)
