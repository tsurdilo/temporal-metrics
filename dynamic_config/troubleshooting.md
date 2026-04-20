# Temporal Dynamic Config — Troubleshooting Patterns

## Table of Contents

- [RESOURCE_EXHAUSTED errors](#resource_exhausted-errors)
- [History cache misses / memory pressure](#history-cache-misses--memory-pressure)
- [ES / Visibility indexing backlog](#es--visibility-indexing-backlog)
- [Too many outstanding task appends](#too-many-outstanding-task-appends)
- [Idle cluster hitting DB too often](#idle-cluster-hitting-db-too-often)
- [History growing too large](#history-growing-too-large)
- [Schedules too slow / rate limited / resource issues](#schedules-too-slow--rate-limited--resource-issues)

---

## Common Troubleshooting Patterns

### RESOURCE_EXHAUSTED errors
- `RpsLimit` cause → adjust `frontend.rps`, `frontend.namespaceRPS`, `frontend.globalNamespaceRPS`
- `ConcurrentLimit` cause → adjust `frontend.namespaceCount`, `frontend.globalNamespaceCount`
- Visibility `list*` operations → adjust `frontend.namespaceRPS.visibility` (default only 10!)
- Persistence overloaded → adjust `frontend.persistenceMaxQPS`, `history.persistenceMaxQPS`, `matching.persistenceMaxQPS`

### History cache misses / memory pressure
- Check `history.cacheMaxSize` (per-shard, default 512) and `history.hostLevelCacheMaxSize` (host-level, default 128K)
- Enable `history.cacheBackgroundEvict` for proactive cleanup
- Tune `history.cacheTTL` / `history.eventsCacheTTL` (default 1h)

### ES / Visibility indexing backlog
- Reduce `history.visibilityProcessorMaxPollRPS` and `history.visibilityProcessorSchedulerWorkerCount` to throttle indexing speed
- Set `history.visibilityTaskWorkerCount: 0` to fully pause indexing during reindex operations
- Set `system.secondaryVisibilityWritingMode: dual` when migrating to ES

### Too many outstanding task appends
- Increase `matching.outstandingTaskAppendsThreshold` (restart required)
- Consider increasing `matching.numTaskqueueWritePartitions` / `matching.numTaskqueueReadPartitions`

### Idle cluster hitting DB too often
- Increase `history.transferProcessorMaxPollInterval`, `history.timerProcessorMaxPollInterval`, `history.visibilityProcessorMaxPollInterval`, `history.shardUpdateMinInterval`

### History growing too large
- Reduce `limit.historySize.suggestContinueAsNew` and `limit.historyCount.suggestContinueAsNew` to encourage continue-as-new earlier

### Schedules too slow / rate limited / resource issues

These three configs **always go together** when tuning schedules:

- `worker.schedulerNamespaceStartWorkflowRPS` — the RPS limit is **per namespace as a whole** (divided evenly among workers if more than one). Default 30. Increase as needed, up to 400–500 is fine. If `schedule_rate_limited` metric is non-zero, you're being throttled here.
- `worker.perNamespaceWorkerCount` — default is 1, meaning **all schedule work for a namespace runs on a single pod**. This is a very common cause of disproportionate CPU/memory on one worker pod. Set to your number of worker pods (or slightly higher).
- `worker.perNamespaceWorkerOptions` — set `MaxConcurrentWorkflowTaskPollers` to **1 per 4 "schedule RPS"** you want. Going higher doesn't hurt. Also set `MaxConcurrentActivityTaskPollers`.

**Example config for ~100 schedules RPS:**
```yaml
worker.schedulerNamespaceStartWorkflowRPS:
  - value: 100
worker.perNamespaceWorkerOptions:
  - value:
      MaxConcurrentWorkflowTaskPollers: 25
      MaxConcurrentActivityTaskPollers: 10
worker.perNamespaceWorkerCount:
  - value: 3
```

**Example config for ~400 schedules RPS:**
```yaml
worker.schedulerNamespaceStartWorkflowRPS:
  - value: 400
worker.perNamespaceWorkerOptions:
  - value:
      MaxConcurrentWorkflowTaskPollers: 100
      MaxConcurrentActivityTaskPollers: 10
worker.perNamespaceWorkerCount:
  - value: 4
```

> **Note:** `worker.perNamespaceWorkerOptions` is a map-valued setting. `DescribeSchedule` slowness can also be a symptom — it relies on `QueryWorkflow` on the scheduler workflow, so check `schedule_to_start` latency on your server worker if queries are slow.