# Temporal Retention Expiry — Implementation Reference

Retention expiry is the process by which closed workflow executions are permanently deleted from
primary storage once the namespace retention period has elapsed. It is implemented as a
**timer queue task** (`DeleteHistoryEventTask`) processed by the history service — there is no
Temporal workflow involved.

---

## Architecture Overview

```
Workflow closes
     │
     ├─── archival enabled? ──YES──► ArchiveExecutionTask fires first (see archival-info.md)
     │                                    │ after archival completes, adds DeleteHistoryEventTask
     │                                    ▼
     └─── archival disabled? ─────► DeleteHistoryEventTask added directly at close time
                                         │
                                         ▼
                              scheduled at: closeTime + retention + jitter(RetentionTimerJitterDuration)
                                         │
                                         ▼
                              Timer queue fires DeleteHistoryEventTask
                              (timer_queue_active_task_executor.go  /
                               timer_queue_standby_task_executor.go)
                                         │
                                         ▼
                              executeDeleteHistoryEventTask()
                              (timer_queue_task_executor_base.go:81)
                                         │
                                         ▼
                              deleteManager.DeleteWorkflowExecutionByRetention()
                              (deletemanager/delete_manager.go:129)
                                         │
                                         ▼
                              shardContext.DeleteWorkflowExecution() — 4-stage deletion
                              (shard/context_impl.go:960)
```

---

## Key Files

| File | Purpose |
|---|---|
| `service/history/workflow/task_generator.go:341` | `GenerateDeleteHistoryEventTask()` — schedules the timer task on workflow close |
| `service/history/tasks/workflow_cleanup_timer.go` | `DeleteHistoryEventTask` struct definition |
| `service/history/timer_queue_task_executor_base.go:81` | `executeDeleteHistoryEventTask()` — shared executor for active and standby |
| `service/history/timer_queue_active_task_executor.go:119` | Routes `DeleteHistoryEventTask` on active namespaces |
| `service/history/timer_queue_standby_task_executor.go:104` | Routes `DeleteHistoryEventTask` on standby namespaces |
| `service/history/deletemanager/delete_manager.go` | `DeleteWorkflowExecutionByRetention()` — top-level delete orchestration |
| `service/history/shard/context_impl.go:960` | `DeleteWorkflowExecution()` — 4-stage idempotent deletion |
| `service/history/workflow/task_refresher.go:89` | Re-generates the task if it needs to be refreshed |

---

## Task Schedule Formula

```
deleteTime = closeTime + namespaceRetention + FullJitter(RetentionTimerJitterDuration)
```

- `closeTime` — time the workflow closed
- `namespaceRetention` — configured on the namespace (floored by `system.namespaceMinRetentionGlobal` / `system.namespaceMinRetentionLocal`)
- `FullJitter(RetentionTimerJitterDuration)` — random value in `[0, history.retentionTimerJitterDuration)` to spread load

**Source:** `service/history/workflow/task_generator.go:356`

---

## Two Paths Into Retention Expiry

| Path | When | How `DeleteHistoryEventTask` is created |
|---|---|---|
| **Direct** (no archival) | Archival disabled at cluster or namespace level | Added by `task_generator.go:267` immediately at workflow close, scheduled at `closeTime + retention + jitter` |
| **Post-archival** | Archival enabled and completes successfully | Added by `archival_queue_task_executor.go:244` after archival finishes, using the workflow's `closeTime` for the same formula |

In both cases the `DeleteHistoryEventTask.VisibilityTimestamp` is the scheduled fire time; the
timer queue will not execute the task before that time.

> **`VisibilityTimestamp` vs the visibility store:** Despite the name, `VisibilityTimestamp` has
> nothing to do with the workflow visibility store (Elasticsearch / SQL visibility). It is simply
> the field name all timer tasks use to indicate their scheduled fire time in the timer queue. A
> flakey visibility store does not affect when `DeleteHistoryEventTask` fires or whether it
> succeeds.
>
> However, a flakey visibility store does affect **Stage 1** of the 4-stage deletion, which
> enqueues a `DeleteExecutionVisibilityTask` to remove the workflow's record from the visibility
> store. If the visibility store is unavailable, that task will retry until the store recovers.
> Stages 2–4 (current execution pointer, mutable state, history branch) are independent tasks and
> proceed normally — they are not blocked by visibility store failures. The only observable effect
> is that the workflow remains visible in search results longer than expected until
> `DeleteExecutionVisibilityTask` eventually succeeds.

---

## Dynamic Configs

### Retention Bounds

| JSON Key | Description |
|---|---|
| `system.namespaceMinRetentionGlobal` | Minimum allowed retention for global (multi-cluster) namespaces |
| `system.namespaceMinRetentionLocal` | Minimum allowed retention for local namespaces |
| `history.retentionTimerJitterDuration` | Max random jitter added to the delete schedule time to spread load across the retention boundary |

### Timer Queue / Processor

| JSON Key | Description |
|---|---|
| `history.timerTaskBatchSize` | Number of timer tasks fetched per poll |
| `history.timerProcessorSchedulerWorkerCount` | Number of goroutine workers in the host-level task scheduler |
| `history.timerProcessorMaxPollRPS` | Max task poll rate (per second) |
| `history.timerProcessorMaxPollHostRPS` | Max task poll rate across all timer processors on a host |
| `history.timerProcessorMaxPollInterval` | Max interval between polls |
| `history.timerProcessorMaxPollIntervalJitterCoefficient` | Jitter coefficient applied to poll interval |
| `history.timerProcessorUpdateAckInterval` | How often the processor updates its ack level |
| `history.timerProcessorUpdateAckIntervalJitterCoefficient` | Jitter coefficient for ack interval |
| `history.timerProcessorPollBackoffInterval` | Backoff interval on empty poll |
| `history.timerProcessorMaxTimeShift` | Max clock skew the timer processor tolerates |
| `history.timerQueueMaxReaderCount` | Max number of concurrent readers in the timer queue |

### Replication

| JSON Key | Description |
|---|---|
| `history.enableDeleteWorkflowExecutionReplication` | Whether a replication task is generated alongside the deletion so remote clusters also delete the execution |

### Secondary Cleanup (Scanner / Scavenger)

These configs control the scanner-based fallback path (see [Secondary Cleanup Path](#secondary-cleanup-path--scavenger-workflows) below).

| JSON Key | Default | Description |
|---|---|---|
| `worker.executionDataDurationBuffer` | `90d` | Extra buffer added on top of namespace retention before the scavenger will delete a workflow. A closed workflow is only considered overdue when `timeSinceLastUpdate > retention + buffer`. |
| `worker.historyScannerDataMinAge` | `60d` | History scavenger skips any history branch whose fork time is newer than this age. Prevents the scavenger from racing with the normal deletion path. |
| `worker.historyScannerVerifyRetention` | `true` | When enabled, the history scavenger also checks if a found mutable state has outlived its retention period and calls `adminservice.DeleteWorkflowExecution` to force-delete it. |
| `worker.historyScannerEnabled` | `true` | Enables the history scavenger workflow (`temporal-sys-history-scanner-workflow`). |
| `worker.executionsScannerEnabled` | `false` | Enables the executions scavenger workflow (`temporal-sys-executions-scanner-workflow`). **No effect when SQL persistence (PostgreSQL, MySQL, SQLite) is used** — SQL support is not yet implemented. |

---

## Service Operations

### Phase 1 — Task Execution Entry Point

**File:** `service/history/timer_queue_task_executor_base.go:81`

| Operation | Detail |
|---|---|
| `cache.GetOrCreateChasmExecution()` | Acquire workflow execution context with low-priority lock |
| `loadMutableStateForTimerTask()` | Load mutable state from persistence |
| Version check (`CheckTaskVersion`) | Verify task version matches namespace failover version; drops stale tasks |
| Guard: workflow still running? | If yes, drop task (cross-DC replication can resurrect a workflow) |
| Guard: mutable state missing? | Falls back directly to `DeleteHistoryBranch` using `task.BranchToken` |
| `deleteManager.DeleteWorkflowExecutionByRetention()` | Dispatch to delete manager |

### Phase 2 — Delete Manager

**File:** `service/history/deletemanager/delete_manager.go:129`

- Pre-marks replication stage as processed (retention deletion skips cross-cluster replication —
  each cluster runs its own independent retention timer)
- Reads `currentBranchToken` and `closeVisibilityTaskId` from mutable state
- Calls `shardContext.DeleteWorkflowExecution()` — the 4-stage deletion
- On success: emits `workflow_cleanup_delete` counter (tagged `operation=ProcessDeleteHistoryEvent`)
- Calls `weCtx.Clear()` to invalidate any cached copy of the execution

### Phase 3 — 4-Stage Idempotent Deletion

**File:** `service/history/shard/context_impl.go:960`

Each stage is checkpointed in `DeleteWorkflowExecutionStage` so partial failures are safely
resumable on retry.

| Stage | Persistence Operation | Detail |
|---|---|---|
| **1** | `AddHistoryTasksRequest` | Enqueues `DeleteExecutionVisibilityTask` to remove the visibility record. Also enqueues `DeleteExecutionReplicationTask` if `history.enableDeleteWorkflowExecutionReplication=true` and namespace is multi-cluster active. Calls `engine.NotifyNewTasks()` after write. |
| **2** | `DeleteCurrentWorkflowExecution` | Removes the current-execution pointer (`executions` table current run entry). |
| **3** | `DeleteWorkflowExecution` | Deletes the workflow mutable state record. |
| **4** | `DeleteHistoryBranch` | Deletes the history branch using `BranchToken` from mutable state. Runs outside the io semaphore (can be large). |

Stages 1–3 are wrapped under a single io semaphore acquisition. Stage 4 is separate.

---

## Metrics

| Metric | Tags | Emitted by | Meaning |
|---|---|---|---|
| `workflow_cleanup_delete` | `operation=ProcessDeleteHistoryEvent`, `namespace=...` | `deletemanager/delete_manager.go:179` | One count per successfully deleted workflow execution via retention |
| Standard timer queue task metrics | `operation=ProcessDeleteHistoryEvent`, `task_type=DeleteHistoryEventTask` | Timer queue infrastructure | Task latency, attempts, failures for `DeleteHistoryEventTask` |

> **Note:** The operation tag `ProcessDeleteHistoryEvent` appears on **both** the archival-triggered
> deletion path and the direct (no-archival) retention path. It cannot alone distinguish the two.
> See `archival-info.md` § Observability for how to separate them.

---

## How to Know Retention Expiry Is Running

Unlike archival, retention expiry has no dedicated observable metrics beyond the general timer queue
infrastructure and `workflow_cleanup_delete`. Useful signals:

| Signal | How to use |
|---|---|
| `workflow_cleanup_delete{operation="ProcessDeleteHistoryEvent"}` rate | Direct count of workflows deleted by retention expiry (includes post-archival deletes) |
| Timer queue task metrics `task_type=DeleteHistoryEventTask` | Throughput, latency, and error rate of the timer task processing |
| `persistence.DeleteCurrentWorkflowExecution` rate | Stage 2 of the deletion — one-to-one with `workflow_cleanup_delete` if no failures |
| `persistence.DeleteHistoryBranch` rate | Stage 4 — also triggered by history scavenger and archival, so cross-reference with other signals |
| Server logs at `WARN` level | `"Dropping archival task because mutable state is nil"` and similar in `timer_queue_task_executor_base.go` |

### Distinguishing direct retention expiry from post-archival deletion

| Indicator | Direct (no archival) | Post-archival |
|---|---|---|
| `archiver_archive_latency` | No preceding spike | Spike occurs ~retention period before `workflow_cleanup_delete` spike |
| `ArchivalQueueProcessor` task metrics | Silent | Active |
| Timing of `DeleteCurrentWorkflowExecution` | Correlates with workflow close rate shifted by retention | Decoupled from close rate; correlates with archival task completions |

---

## Secondary Cleanup Path — Scavenger Workflows

The primary path (`DeleteHistoryEventTask` timer) handles normal deletion. The scanners are a
**secondary safety net** that catches data the primary path missed — due to bugs, crashes, or
partial failures. They run as long-lived Temporal workflows in the `temporal-system` namespace.

### Backend coverage at a glance

| Cleanup path | Cassandra | PostgreSQL / MySQL / SQLite (SQL) |
|---|---|---|
| Primary timer (`DeleteHistoryEventTask`) | Yes | Yes |
| History scavenger (`temporal-sys-history-scanner-workflow`) | Yes | Yes |
| Executions scavenger (`temporal-sys-executions-scanner-workflow`) | Yes (when enabled) | **No — SQL not implemented** |

### Why this exists

The primary timer task can be lost or never fire if:
- The shard was unavailable when the task was due
- A bug caused the task not to be created
- The history branch was orphaned (mutable state deleted but history branch left behind)
- Archival failed mid-way, leaving a partially deleted execution

---

### History Scavenger (`temporal-sys-history-scanner-workflow`)

**File:** `service/worker/scanner/history/scavenger.go`
**Backends:** ALL (Cassandra, PostgreSQL, MySQL, SQLite)

One full pass over every history branch in the system:

1. Calls `persistence.GetAllHistoryTreeBranches` (paginated, 100 per page) to enumerate every history branch stored across all shards.
2. **Skips** any branch whose `ForkTime` is newer than `worker.historyScannerDataMinAge` (default 60d) — prevents racing with the normal deletion path. Emits `history_scavenger_skip`.
3. For each remaining branch, calls `historyservice.DescribeMutableState`:
   - **Mutable state not found** (`NotFound` / `NamespaceNotFound`): the branch is an orphan — mutable state was deleted but the history branch was not. Calls `persistence.DeleteHistoryBranch` directly to clean it up.
   - **Mutable state found** and `worker.historyScannerVerifyRetention=true`: checks whether the closed workflow has outlived its retention period:
     ```
     timeSinceLastUpdate > namespaceRetention + worker.executionDataDurationBuffer
     ```
     If yes, calls `adminservice.DeleteWorkflowExecution` to force-delete the entire execution. Errors are **logged as warnings and not retried** — best-effort so one failure doesn't block the whole scan.

**Operations called:**

| Operation | Detail |
|---|---|
| `persistence.GetAllHistoryTreeBranches` | Paginated scan of all history branches |
| `historyservice.DescribeMutableState` | Check whether mutable state still exists for a branch |
| `persistence.DeleteHistoryBranch` | Delete orphaned history branch (no mutable state found) |
| `adminservice.DeleteWorkflowExecution` | Force-delete a workflow that outlived `retention + buffer` |

**Metrics:** `history_scavenger_success`, `history_scavenger_error`, `history_scavenger_skip` (tagged `operation=HistoryScavengerScope`)

---

### Executions Scavenger (`temporal-sys-executions-scanner-workflow`)

**File:** `service/worker/scanner/executions/scavenger.go`
**Backends: Cassandra ONLY. Does not run on SQL persistence (PostgreSQL, MySQL, SQLite).**

The dynamic config `worker.executionsScannerEnabled` (default `false`) has no effect on SQL
backends. The config docstring states explicitly: *"This flag has no effect when SQL persistence is
used, because executions scanner support for SQL is not yet implemented."*
(`common/dynamicconfig/constants.go:3125`)

**What it does (Cassandra only, when enabled):**

Iterates over all workflow executions across every shard (open and closed) and validates each one
using `mutable_state_validator.go`. For closed executions it applies the same buffer check as the
history scavenger:

```
timeSinceLastUpdate > namespaceRetention + worker.executionDataDurationBuffer
```

If a workflow has outlived the buffer, it is flagged for deletion via the history service. Unlike
the history scavenger, this operates at the execution level (not the history branch level), so it
can catch cases where mutable state still exists but should have been deleted.

---

### The `worker.executionDataDurationBuffer` (default 90 days)

This is an extra safety margin on top of namespace retention used **only by the scavengers**. The
scavengers will not delete a workflow until at least `retention + 90d` have elapsed since its last
update — regardless of what the namespace retention period says.

**This config does not affect the primary timer path.** The `DeleteHistoryEventTask` timer fires
at `closeTime + retention + jitter` and ignores this buffer entirely.

| Backend | Primary timer | History scavenger | Executions scavenger |
|---|---|---|---|
| Cassandra | Fires at `closeTime + retention + jitter` | Fires at `lastUpdate + retention + 90d` | Fires at `lastUpdate + retention + 90d` |
| PostgreSQL / MySQL / SQLite | Fires at `closeTime + retention + jitter` | Fires at `lastUpdate + retention + 90d` | **Does not run** |

---

## Cross-Cluster Behavior

- Each cluster runs its **own independent retention timer**. There is no cross-cluster coordination
  for retention expiry.
- `DeleteWorkflowExecutionByRetention` pre-marks the replication stage as processed
  (`deletemanager/delete_manager.go:139`) so no replication task is sent for the deletion itself.
- If `history.enableDeleteWorkflowExecutionReplication=true`, a `DeleteExecutionReplicationTask`
  **is** generated in Stage 1 to explicitly propagate the delete to passive clusters rather than
  relying on their independent timers.
- On standby clusters, `DeleteHistoryEventTask` is handled by
  `timer_queue_standby_task_executor.go:104` which calls the same base executor.
