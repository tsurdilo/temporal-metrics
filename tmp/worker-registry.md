# Temporal Worker Registry — Metrics and Storage Reference


## 1. What the Worker Registry Is

The worker registry is an **in-memory, sharded LRU cache** inside the **matching service** that stores worker heartbeat data. It is the backing store for all four `worker_registry_*` metrics. Nothing is persisted to a database — the registry lives entirely in matching service memory and is rebuilt from incoming heartbeats.

SDK workers only communicate with the **frontend service**. Workers call `WorkflowService.RecordWorkerHeartbeat` on the frontend, which resolves the namespace name to a namespace ID and forwards the request to the matching service via an internal gRPC call (`matchingservice.RecordWorkerHeartbeat`). The matching service handler then writes the heartbeat into the registry, which maintains per-namespace, per-worker-instance state.

### Routing note

Unlike task-queue operations, heartbeat routing to matching does not use a task queue name. The matching client uses `"not-applicable"` as a placeholder task queue and routes purely by namespace ID. This means all heartbeats for a given namespace are consistently sent to the same matching service instance.

---

## 2. Metric Definitions

All four metrics are defined in `common/metrics/metric_defs.go` (~line 1244):

| Metric | Type | Constant | Description |
|---|---|---|---|
| `worker_registry_workers_added` | Counter | `WorkerRegistryWorkersAdded` | Count of new workers registered (first heartbeat seen) |
| `worker_registry_workers_removed` | Counter | `WorkerRegistryWorkersRemoved` | Count of workers removed (shutdown status or eviction) |
| `worker_registry_capacity_utilization` | Gauge | `WorkerRegistryCapacityUtilizationMetric` | Ratio of current entries to `maxItems` |
| `worker_registry_activity_slots_used` | Dimensionless Histogram | `WorkerRegistryActivitySlotsUsed` | `CurrentUsedSlots` from `ActivityTaskSlotsInfo` per heartbeat |

---

## 3. In-Memory Data Structure

**File:** `service/matching/workers/registry_impl.go`

### Top-level registry (`registryImpl`)

```go
type registryImpl struct {
    buckets            []*bucket                        // sharded keyspace
    maxItemsFn         dynamicconfig.IntPropertyFn      // dynamic max entries
    ttlFn              dynamicconfig.DurationPropertyFn // entry TTL
    minEvictAgeFn      dynamicconfig.DurationPropertyFn // minimum age before capacity eviction
    evictionIntervalFn dynamicconfig.DurationPropertyFn // how often eviction loop runs
    total              atomic.Int64                     // thread-safe entry count
    quit               chan struct{}
    seed               maphash.Seed
    metricsHandler     metrics.Handler
    metricsEmitter     *workerMetricsEmitter
}
```

### Shard unit (`bucket`)

```go
type bucket struct {
    mu         sync.Mutex
    namespaces map[namespace.ID]map[string]*entry  // nsID → instanceKey → entry
    order      *list.List                          // LRU chain: front=oldest, back=newest
}
```

### Worker record (`entry`)

```go
type entry struct {
    nsID           namespace.ID
    hb             *workerpb.WorkerHeartbeat  // the full heartbeat proto
    lastSeen       time.Time
    elem           *list.Element              // position in LRU list
    isSystemWorker bool
}
```

### WorkerHeartbeat proto fields relevant to metrics

| Field | Type | Used by |
|---|---|---|
| `WorkerInstanceKey` | string | Registry key (unique worker identifier) |
| `Status` | enum | `WORKER_STATUS_SHUTDOWN` triggers removal |
| `ActivityTaskSlotsInfo.CurrentUsedSlots` | int32 | `worker_registry_activity_slots_used` histogram |
| `TaskQueue` | string | Tag context for other metrics |

---

## 4. Sharding

Bucket selection is by `hash(namespace)`. All workers in the same namespace land in the same bucket. Each bucket has its own mutex so contention is bounded to per-namespace concurrency. The number of buckets is configured via `WorkerRegistryNumBuckets`.

---

## 5. Metric Emission — Full Map

### `worker_registry_workers_added`

- **File:** `registry_impl.go`
- **Method:** `upsertHeartbeats()` (~line 267)
- **Trigger:** A heartbeat arrives for an instance key not yet in the registry (new entry created)
- **Value:** Count of newly inserted entries in the batch

### `worker_registry_workers_removed`

Emitted from three places:

| Location | Trigger |
|---|---|
| `upsertHeartbeats()` ~line 270 | Worker sends heartbeat with `WORKER_STATUS_SHUTDOWN` |
| `evictByTTL()` ~line 335 | Entry's `lastSeen` is older than TTL |
| `evictByCapacity()` ~line 359 | Registry is over `maxItems` and entry is old enough to evict |

### `worker_registry_capacity_utilization`

- **Method:** `recordUtilizationMetric()` (~line 276)
- **Formula:** `float64(m.total.Load()) / float64(m.maxItemsFn())`
- **Emitted:** After every upsert and after every eviction loop tick
- **Range:** 0.0–1.0+; can briefly exceed 1.0 before capacity eviction catches up

### `worker_registry_activity_slots_used`

- **File:** `worker_metrics_emitter.go`
- **Method:** `workerMetricsEmitter.emit()` (~line 37)
- **Trigger:** Called on every `RecordWorkerHeartbeats()` call, after the upsert
- **Value:** `hb.ActivityTaskSlotsInfo.CurrentUsedSlots` cast to int64
- **Condition:** Only emitted if the heartbeat includes a non-nil `ActivityTaskSlotsInfo`

---

## 6. Data Flow

```
SDK worker gRPC call: WorkflowService.RecordWorkerHeartbeat
  → frontend WorkflowHandler.RecordWorkerHeartbeat()  (workflow_handler.go:7172)
      resolves namespace name → namespace ID
      forwards via internal gRPC to matching service
  → matching Handler.RecordWorkerHeartbeat()           (handler.go:592)
      extracts principal from context headers
    → registryImpl.RecordWorkerHeartbeats()            (registry_impl.go)
        ├─ upsertHeartbeats()
        │    ├─ hash(namespace) → select bucket
        │    ├─ SHUTDOWN status? → remove entry, incr removed
        │    ├─ Key exists?      → update hb + lastSeen, move to LRU back
        │    └─ Key new?         → create entry, push to LRU back, incr added
        │
        │    emit: workers_added (if added > 0)
        │    emit: workers_removed (if removed > 0)
        │    emit: capacity_utilization
        │
        └─ metricsEmitter.emit()
             └─ for each heartbeat:
                  if ActivityTaskSlotsInfo != nil:
                    emit: activity_slots_used (CurrentUsedSlots)

Background eviction loop (every EvictionInterval):
  → evictByTTL()
  │    removes entries where lastSeen < now - TTL
  │    emit: workers_removed, capacity_utilization
  │
  └─ evictByCapacity()
       while total > maxItems:
         evict oldest entry with lastSeen > minEvictAge
         emit: workers_removed, capacity_utilization
```

---

## 7. Eviction Mechanics

### TTL eviction (`evictByTTL`)

Scans the front of each bucket's LRU list (oldest entries). Removes any entry where `lastSeen < now - TTL`. Runs every `EvictionInterval`.

### Capacity eviction (`evictByCapacity`)

Runs when `total > maxItems`. Iterates buckets round-robin for fairness. Will not evict an entry younger than `minEvictAge` — this prevents churn on workers that just registered. If no eligible entry is found in a full round, the loop breaks (eviction blocked by age). There is a separate `worker_registry_eviction_blocked_by_age` metric that signals this condition.

### `minEvictAge` interaction

`minEvictAge` is the safety valve. If all entries are too young to evict, the registry can temporarily exceed `maxItems`. `capacity_utilization` will be > 1.0 in this state. This is expected behavior during sudden worker registration bursts.

---

## 8. Configuration (Dynamic Config, Matching Service)

All values are set via dynamic config and wired through `service/matching/fx.go` (~line 210). String names and defaults are from `common/dynamicconfig/constants.go` (~line 1494).

| Go constant | YAML config key | Default | Description |
|---|---|---|---|
| `MatchingWorkerRegistryNumBuckets` | `matching.workerRegistryNumBuckets` | `10` | Number of buckets used to partition the registry keyspace for reduced lock contention. **Changes require a restart.** |
| `MatchingWorkerRegistryMaxEntries` | `matching.workerRegistryMaxEntries` | `1000000` | Maximum total worker heartbeat entries across all namespaces. When exceeded, oldest entries (older than `MinEvictAge`) are evicted. Drives the `capacity_utilization` denominator. |
| `MatchingWorkerRegistryEntryTTL` | `matching.workerRegistryEntryTTL` | `5m` | Time after which an entry with no new heartbeat is considered expired and eligible for TTL eviction. Workers typically heartbeat every 30–60s, so 5 minutes without a heartbeat indicates a likely dead worker. |
| `MatchingWorkerRegistryMinEvictAge` | `matching.workerRegistryMinEvictAge` | `1m` | Minimum age of an entry before it can be evicted due to capacity pressure. Prevents evicting recently-heartbeated workers when the registry is at capacity. Lower values help handle crash-looping workers more aggressively. |
| `MatchingWorkerRegistryEvictionInterval` | `matching.workerRegistryEvictionInterval` | `1m` | How often the background eviction loop runs. Should be shorter than `EntryTTL` for timely cleanup. Lower values mean faster cleanup at the cost of more CPU overhead. |

---

## 9. Operational Notes

- **No persistence.** The registry is fully rebuilt from worker heartbeats. A matching service restart means zero entries until workers re-heartbeat.
- **Per matching host.** Each matching service instance has its own registry. There is no cross-host synchronization.
- **`workers_added` vs. `workers_removed` imbalance** is normal — removed includes both shutdown and eviction; added only fires on first registration per instance key.
- **`activity_slots_used` gaps** mean the worker is not sending `ActivityTaskSlotsInfo` in its heartbeat. This is SDK-version dependent — older SDKs may not populate this field.
- **`capacity_utilization` > 1.0** indicates the registry briefly exceeded `maxItems`, blocked from eviction by `minEvictAge`. Sustained values > 1.0 warrant increasing `WorkerRegistryMaxEntries` or decreasing `WorkerRegistryMinEvictAge`.

---

## 10. How Heartbeats Are Routed to Matching Instances

### Routing key construction

SDK workers only talk to the frontend. When the frontend forwards a heartbeat to matching, it builds a partition using `"not-applicable"` as a placeholder task queue name and the namespace ID as the only meaningful input (`client/matching/client_gen.go`):

```go
tqid.NormalPartitionFromRpcName("not-applicable", namespaceId, UNSPECIFIED)
```

This produces a routing key string of the form `"<namespace-id>:not-applicable:0"`. The task queue name carries no meaning here — namespace ID is the entire basis for routing.

### Consistent hash ring

That routing key is fed into a **consistent hash ring** backed by the `ringpop-go` library using the Farm Fingerprint32 hash function (`common/membership/ringpop/service_resolver.go`). The ring holds all live matching hosts. The key hashes to one position on the ring and whichever host owns that position receives the request.

This is fully deterministic: the same namespace ID always produces the same hash and therefore always resolves to the same matching host, as long as ring membership has not changed.

### Load distribution across matching hosts

There is no explicit load balancer. Distribution falls out naturally from consistent hashing — namespace IDs are essentially random strings, their hashes spread across the ring, and each matching host ends up owning roughly `1/N` of all namespaces (where N = number of matching instances). In a 5-host matching fleet, each host carries the worker registry for approximately 20% of namespaces. The `matching.workerRegistryMaxEntries` limit (default 1,000,000) is per host and applies only to whichever namespaces that host owns.

### What happens when matching hosts join or leave

Ringpop detects membership changes via the SWIM gossip protocol and rebuilds the ring (`service_resolver.go`). With consistent hashing, only the namespaces that were owned by the departed or newly joined host are redistributed — all other namespace-to-host mappings remain unchanged. A single matching host restart disrupts the registry only for the namespaces it owned; those registries are empty until workers for those namespaces re-heartbeat.

### Implication for registry metrics

Because each matching instance holds an independent registry for its slice of namespaces, `worker_registry_*` metrics are per-host. `capacity_utilization` and entry counts reflect only the namespaces owned by that instance, not the cluster total. To get a cluster-wide view, aggregate across all matching instances — which is what the `quantile()` expressions in the dashboard do.
