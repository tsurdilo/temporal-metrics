# Change Temporal Task Queue Partitions — Guide

## What Are Task Queue Partitions?

Task queues in Temporal have multiple partitions. Partitions belonging to the same task queue can be owned by different matching hosts. The default number of partitions per task queue is 4.

The partition count is controlled via two dynamic configs:

| Config | Role |
|---|---|
| `matching.numTaskqueueWritePartitions` | Controls how many partitions new tasks are written to |
| `matching.numTaskqueueReadPartitions` | Controls how many partitions workers poll from |

These are set per task queue and take effect without a server restart.

---

## Defaults and Sizing

- **Default: 4 partitions** — sufficient for the vast majority of use cases.
- For high throughput use cases, 16 partitions is typically sufficient. There are cases where more may be needed — see the sections below on how to identify when that's the case.
- For low-traffic task queues, reducing partitions down to 1 can be beneficial to reduce resource consumption on matching hosts.
- Setting partitions to 1 is also useful when strict and exact activity task dispatch rate control is needed — with multiple partitions, rate limits are applied per partition, making it harder to enforce precise cluster-wide dispatch rates for a given task queue.

> Only increase partitions if you have confirmed evidence that matching is the bottleneck.

---

## When to Increase Partitions

### Signals to look for

**High matching service CPU** — if the rate of tasks keeps growing, partitions may become a bottleneck. Check your matching service CPU — if it is running high, matching may be the bottleneck.

**`task_write_throttle_count`** — matching is falling behind writing tasks to persistence. Note: this is usually caused by low sync match rate due to not having enough workers, so rule that out first.

**`sync_throttle_count`** — tasks are being throttled by matching. The default limit is 1,000 per second per task queue partition. If you are hitting this consistently, more partitions can help distribute the load.

**`asyncmatch_latency`** — measures the time from task creation to delivery. The larger this latency, the longer tasks are sitting in the queue waiting for workers to pick them up. If workers are healthy, high async match latency can indicate matching is the bottleneck.

---

## How to Increase Partitions (Safe)

Increasing is safe — you can raise both configs simultaneously to the target value.

The config supports a global default and per-task-queue overrides in the same file:

```yaml
matching.numTaskqueueWritePartitions:
  # Global default for all task queues
  - value: 4
    constraints: {}
  # Override for a specific task queue in a specific namespace
  - value: 8
    constraints: {
      namespace: 'your-namespace',
      taskQueueName: 'your-high-traffic-tq'
    }

matching.numTaskqueueReadPartitions:
  # Global default for all task queues
  - value: 4
    constraints: {}
  # Override for a specific task queue in a specific namespace
  - value: 8
    constraints: {
      namespace: 'your-namespace',
      taskQueueName: 'your-high-traffic-tq'
    }
```

Both can be changed at the same time. New tasks will immediately start being distributed across the additional partitions.

---

## How to Decrease Partitions (Requires Care)

Decreasing is **not safe to do all at once**. If you reduce read partitions before the extra partitions are drained, tasks sitting in those partitions will be stranded and never picked up by workers.

### Required order

**Step 1 — Decrease write partitions first**

Lower `matching.numTaskqueueWritePartitions` to the target value. No new tasks will be written to the partitions being retired.

```yaml
matching.numTaskqueueWritePartitions:
  - value: 4  # reduced target — global default
    constraints: {}
  - value: 4  # reduced target — specific task queue
    constraints: {
      namespace: 'your-namespace',
      taskQueueName: 'your-task-queue'
    }
# Leave numTaskqueueReadPartitions unchanged at the higher value
```

**Step 2 — Wait for drain**

Keep read partitions at the higher value and monitor `asyncmatch_latency`. When async match latency drops to normal levels and stays there, the retired partitions have drained their backlog.

There is currently no built-in metric or signal that explicitly confirms a partition is fully drained — monitoring `asyncmatch_latency` going quiet is the best available indicator.

**Step 3 — Decrease read partitions**

Only after you are confident the extra partitions are drained, lower `matching.numTaskqueueReadPartitions` to match the write partition count.

```yaml
matching.numTaskqueueReadPartitions:
  - value: 4  # now matches write partition count — global default
    constraints: {}
  - value: 4  # now matches write partition count — specific task queue
    constraints: {
      namespace: 'your-namespace',
      taskQueueName: 'your-task-queue'
    }
```

### Summary

```
INCREASE: change both configs together ✅

DECREASE:
  1. Lower numTaskqueueWritePartitions
  2. Wait — monitor asyncmatch_latency until quiet
  3. Lower numTaskqueueReadPartitions
```



---

## Key Metrics Reference

| Metric | What it tells you |
|---|---|
| `asyncmatch_latency` | Time from task creation to worker pickup — high value = tasks sitting in queue |
| `sync_throttle_count` | Tasks throttled by matching — hitting 1000/s/partition limit |
| `task_write_throttle_count` | Matching falling behind writing tasks to persistence |
| Matching host CPU | Direct signal of matching service saturation |

These metrics are available in the **Task Queue Partitions** section of the OSS Server Overview Grafana dashboard.

For additional matching task queue metrics, see the [Matching Task Queue Info section](https://github.com/tsurdilo/temporal-server-operations/blob/main/metrics/dashboards/server/temporal-server-readme.md#12-matching-task-queue-info) of the [Temporal Server Operations dashboard readme](https://github.com/tsurdilo/temporal-server-operations/blob/main/metrics/dashboards/server/temporal-server-readme.md).