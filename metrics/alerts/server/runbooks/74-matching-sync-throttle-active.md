## Matching Partition Sync Throttle Active

**Severity:** Critical
**Component:** matching
**Dashboard panel:** [Sync Throttle Count](https://github.com/tsurdilo/temporal-metrics/blob/main/metrics/dashboards/server/temporal-server-readme.md) — panel ID 403

### What this alert detects

Any non-zero rate of `sync_throttle_count{service_name="matching"}` per namespace and task type for 1 consecutive minute. Fires when any matching partition hits its sync match dispatch limit.

### Why it matters

`sync_throttle_count` increments when the matching service cannot dispatch a task synchronously because the per-partition dispatch rate limit has been reached. This means pollers are waiting but tasks are being held back by the partition limit. If this fires without alert 57 (All Pollers Disconnected) also firing, the task queues need more partitions to distribute the load.

### Triage steps

1. Open the **Sync Throttle Count** panel (panel 403) in the Temporal Server dashboard — confirm which namespace and task type are affected
2. Check if alert 57 (All Pollers Disconnected) is also firing — if so the throttling is a symptom of no workers, not a partition sizing issue
3. Check the current number of task queue partitions for the affected task queue — the default is 1 per partition for regular workflow and activity task queues; sticky task queues always use 1 partition and cannot be increased
4. Check **Async Match Latency** and **Sync Match Latency** panels — elevated match latency alongside throttling confirms dispatch is backed up

### Relevant dynamic config

- `matching.numTaskqueueWritePartitions` — number of write partitions per task queue (default 1); increase for high-throughput task queues to distribute dispatch load across more partitions
- `matching.numTaskqueueReadPartitions` — number of read partitions (default 1); should match write partitions
- `matching.syncMatchWaitDuration` — how long the matching service waits for a sync match before falling back to async (default 200ms); reducing this lowers the window during which throttling can occur but increases async task persistence load
