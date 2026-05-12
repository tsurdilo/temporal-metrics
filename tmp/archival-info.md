# Temporal Archival — Implementation Reference


---

## Architecture Overview

```
Workflow closes
     │
     ▼
task_generator.go — generates ArchiveExecutionTask
     │  scheduled at: closeTime + jitter(ArchivalProcessorArchiveDelay), capped at namespace retention
     ▼
archival_queue_factory.go — archival queue picks up the task
     │
     ▼
archival_queue_task_executor.go — processArchiveExecutionTask()
     │  1. load mutable state
     │  2. build archival.Request (history URI + visibility URI from namespace config)
     │  3. gate on cluster-level archival config (static config)
     │  4. call archiver.Archive()
     │  5. add DeleteHistoryEventTask
     ▼
archival/archiver.go — Archive()
     │  concurrently archives history and/or visibility
     ├── historyArchiver.Archive()   →  filestore / gcloud / s3store
     └── visibilityArchiver.Archive() → filestore / gcloud / s3store
```

After archival, a `DeleteHistoryEventTask` (timer task) triggers a 4-stage deletion of the
workflow from primary storage.

---

## Key Files

| File | Purpose |
|---|---|
| `service/history/workflow/task_generator.go:260` | Generates `ArchiveExecutionTask` on workflow close |
| `service/history/archival_queue_factory.go` | Archival queue factory and scheduler configuration |
| `service/history/archival_queue_task_executor.go` | Queue task executor — orchestrates the archival flow |
| `service/history/archival/archiver.go` | Archives history and visibility to configured backends |
| `service/history/tasks/archive_execution_task.go` | `ArchiveExecutionTask` struct |
| `common/archiver/history_iterator.go` | Reads history events from persistence in paginated batches |
| `common/archiver/filestore/history_archiver.go` | Filestore history archiver |
| `common/archiver/filestore/visibility_archiver.go` | Filestore visibility archiver |
| `common/archiver/gcloud/history_archiver.go` | Google Cloud Storage history archiver |
| `common/archiver/gcloud/visibility_archiver.go` | Google Cloud Storage visibility archiver |
| `common/archiver/s3store/history_archiver.go` | S3 history archiver |
| `common/archiver/s3store/visibility_archiver.go` | S3 visibility archiver |
| `common/archiver/archival_metadata.go` | Cluster-level archival config (static config gate) |
| `common/archiver/provider/provider.go` | Archiver provider — selects backend by URI scheme |
| `service/history/deletemanager/delete_manager.go` | Orchestrates 4-stage workflow deletion post-archival |

---

## Dynamic Configs

All keys are read from cluster dynamic config. URI configuration lives in static config
(`archival.history.state`, `archival.visibility.state`) and in per-namespace settings.

### Archival Queue / Processor

| JSON Key | Description |
|---|---|
| `history.archivalTaskBatchSize` | Number of archival tasks fetched per poll |
| `history.archivalProcessorMaxPollRPS` | Max task poll rate (per second) |
| `history.archivalProcessorMaxPollHostRPS` | Max task poll rate across the host |
| `history.archivalProcessorSchedulerWorkerCount` | Number of goroutine workers in the task scheduler |
| `history.archivalProcessorMaxPollInterval` | Max interval between polls |
| `history.archivalProcessorMaxPollIntervalJitterCoefficient` | Jitter coefficient applied to poll interval |
| `history.archivalProcessorUpdateAckInterval` | How often the processor updates its ack level |
| `history.archivalProcessorUpdateAckIntervalJitterCoefficient` | Jitter coefficient for ack interval |
| `history.archivalProcessorPollBackoffInterval` | Backoff interval on empty poll |
| `history.archivalProcessorArchiveDelay` | Max random delay added to task schedule time to spread load (default: `5m`) |
| `history.archivalBackendMaxRPS` | Rate limit on calls to the archival backend (default: `10000`) |
| `history.archivalQueueMaxReaderCount` | Max number of concurrent readers in the archival queue (default: `2`) |

### System-Level Archival State

| JSON Key | Description |
|---|---|
| `system.historyArchivalState` | Cluster-level history archival state (`"enabled"` / `"disabled"`) |
| `system.enableReadFromHistoryArchival` | Whether reads can be served from history archival |
| `system.visibilityArchivalState` | Cluster-level visibility archival state (`"enabled"` / `"disabled"`) |
| `system.enableReadFromVisibilityArchival` | Whether reads can be served from visibility archival |

### Frontend

| JSON Key | Description |
|---|---|
| `frontend.visibilityArchivalQueryMaxPageSize` | Max page size for `ListArchivedWorkflowExecutions` queries |

---

## Service Operations

### On Workflow Close — Task Scheduling

**File:** `service/history/workflow/task_generator.go:245`

- Checks `archivalEnabled()` — gates on cluster static config AND namespace archival state
- Calls `getRetention()` — reads namespace retention period
- Creates `tasks.ArchiveExecutionTask` with `VisibilityTimestamp = closeTime + jitter`
- Adds to mutable state close tasks via `mutableState.AddTasks()`

---

### Phase 1 — Read & Archive

**File:** `service/history/archival_queue_task_executor.go:96`

| Operation | Detail |
|---|---|
| `weContext.LoadMutableState()` | Load workflow mutable state from persistence |
| `mutableState.GetWorkflowCloseTime()` | Read close timestamp |
| `mutableState.GetCurrentBranchToken()` | Read history branch token |
| `relocatableAttributesFetcher.Fetch()` | Fetch memo and search attributes |
| `carchiver.NewURI(historyURIString)` | Parse history archival URI from namespace config |
| `carchiver.NewURI(visibilityURIString)` | Parse visibility archival URI from namespace config |
| `archiver.Archive(ctx, request)` | Dispatch to backend archiver (concurrent for history + visibility) |

**File:** `service/history/archival/archiver.go:107`

| Operation | Detail |
|---|---|
| `rateLimiter.WaitN(ctx, numTargets)` | Rate-limit archival backend calls (`history.archivalBackendMaxRPS`) |
| `archiverProvider.GetHistoryArchiver(scheme)` | Select backend by URI scheme (`file`, `gs`, `s3`) |
| `historyArchiver.Archive(ctx, uri, request)` | Write history events to archival store |
| `archiverProvider.GetVisibilityArchiver(scheme)` | Select backend by URI scheme |
| `searchAttributeProvider.GetSearchAttributes(indexName)` | Resolve SA types for stringification |
| `searchattribute.Stringify(searchAttributes, saTypeMap)` | Convert SA values to strings |
| `visibilityArchiver.Archive(ctx, uri, record)` | Write visibility record to archival store |

**File:** `common/archiver/history_iterator.go`

| Operation | Detail |
|---|---|
| `persistence.ReadFullPageEventsByBatch()` | Reads history events in pages of 250 via `ReadHistoryBranchRequest` (BranchToken, MinEventID, MaxEventID, ShardID) |

---

### Phase 2 — Deletion (triggered by `DeleteHistoryEventTask` timer task)

After archival completes, a `DeleteHistoryEventTask` is enqueued. When it fires (after retention
expires) the 4-stage deletion runs:

**File:** `service/history/deletemanager/delete_manager.go` → `service/history/shard/context_impl.go`

| Stage | Persistence Operation | Detail |
|---|---|---|
| 1 | `AddHistoryTasksRequest` | Enqueues `DeleteExecutionVisibilityTask` (and `DeleteExecutionReplicationTask` if multi-cluster) |
| 2 | `DeleteCurrentWorkflowExecution` | Removes the current execution pointer from persistence |
| 3 | `DeleteWorkflowExecution` | Deletes the workflow mutable state record |
| 4 | `DeleteHistoryBranch` | Deletes the history branch (using BranchToken from mutable state) |

Each stage is checkpointed so partial failures are resumable. If mutable state is already gone at
deletion time, the executor falls back directly to `DeleteHistoryBranch` via:
`shardContext.GetExecutionManager().DeleteHistoryBranch()`.

Emits metric: `metrics.WorkflowCleanupDeleteCount` on successful deletion.

---

## Gating: When Does Archival Run?

Archival runs only when **both** of the following are true:

1. **Cluster-level** (static config): `archival.history.state = "enabled"` or
   `archival.visibility.state = "enabled"` — checked via
   `shardContext.GetArchivalMetadata().GetHistoryConfig().ClusterConfiguredForArchival()`

2. **Namespace-level**: `namespaceEntry.HistoryArchivalState().State == ARCHIVAL_STATE_ENABLED`
   (and the URI is set on the namespace)

If either condition fails for a target (history or visibility), that target is skipped. If no
targets remain, archival is skipped entirely and a `DeleteHistoryEventTask` is added directly.

---

## Archival Backends

| URI Scheme | Backend | History | Visibility |
|---|---|---|---|
| `file://` | Local filesystem | `common/archiver/filestore/history_archiver.go` | `common/archiver/filestore/visibility_archiver.go` |
| `gs://` | Google Cloud Storage | `common/archiver/gcloud/history_archiver.go` | `common/archiver/gcloud/visibility_archiver.go` |
| `s3://` | AWS S3 | `common/archiver/s3store/history_archiver.go` | `common/archiver/s3store/visibility_archiver.go` |

---

## Observability — Attributing a `DeleteCurrentWorkflowExecution` Spike to Archival

### The problem

Both the archival path and the plain retention-expiry path call the same
`DeleteWorkflowExecutionByRetention` → 4-stage deletion chain, tagged with the same operation scope
(`ProcessDeleteHistoryEvent`). So `DeleteCurrentWorkflowExecution` alone does not point to archival.

### Metric signals exclusive to archival

These metrics are emitted **only** on the archival code path and are never emitted by plain
retention expiry:

| Metric | Source | What it tells you |
|---|---|---|
| `archiver_archive_latency{status="ok"\|"err"\|"rate_limit_exceeded"}` | `service/history/archival/archiver.go:134` | One emission per `ArchiveExecutionTask` processed |
| `archiver_archive_target_latency{target="history"\|"visibility", status=...}` | `service/history/archival/archiver.go:268` | One per archival target per task |
| `history_archiver_archive_success` | `common/archiver/gcloud/history_archiver.go:183` | Successful history archival (gcloud backend) |
| `history_archiver_archive_non_retryable_error` | `common/archiver/` backends | Permanent failure in history archival |
| `history_archiver_archive_transient_error` | `common/archiver/` backends | Retryable failure in history archival |
| `visibility_archiver_archive_non_retryable_error` | `common/archiver/` backends | Permanent failure in visibility archival |
| `archival_task_invalid_uri{namespace=..., failure=...}` | `service/history/archival_queue_task_executor.go:158,176` | Bad URI configured on namespace |

The archival queue processor also tags all its queue-level metrics (task latency, errors, etc.)
with `operation=ArchivalQueueProcessor` and `task_type=ArchivalTaskArchiveExecution` — both
exclusive to the archival path.

### Correlation strategy

```
archiver_archive_latency rate spikes
        │
        │  delay ≈ namespace retention period
        │  (ArchiveExecutionTask adds DeleteHistoryEventTask after archival completes)
        ▼
DeleteCurrentWorkflowExecution spikes  ←  tagged operation=ProcessDeleteHistoryEvent
```

If `archiver_archive_latency` spikes and `DeleteCurrentWorkflowExecution` spikes after a delay
roughly equal to the namespace retention period, archival is the cause. Without archival,
`DeleteCurrentWorkflowExecution` spikes are coupled to workflow close rate with no preceding
archival metrics.

### What you've lost vs. the old workflow-based approach

| Need | Old workflow approach | Current queue-task approach |
|---|---|---|
| Is archival running now? | `ListWorkflowExecutions` on system namespace | Watch `archiver_archive_latency` rate |
| How many tasks queued? | Workflow count query | No direct metric; infer from queue lag |
| Did a specific workflow get archived? | `DescribeWorkflow` on archival workflow | Server logs only — `archiver.go:131` logs on failure, nothing on success |
| Which workflows were archived? | Query archival workflows | No query mechanism; only the archival store (filestore/S3/GCS) holds the record |
