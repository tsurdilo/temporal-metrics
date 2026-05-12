# Changelog — Temporal History Host Health Dashboard

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
