# Worker Service Internal Workflows

All internal workflows, activity types, and dynamic configs for workflows running in `/service/worker`.

---

## Workflow Types

| Component | Workflow Type Name | Task Queue | Namespace | Description | File |
|---|---|---|---|---|---|
| Dummy | `temporal-sys-dummy-workflow` | `temporal-sys-per-ns-tq` | per-namespace | Sentinel workflow that hangs open for a fixed duration to reserve a workflow ID-space key and prevent V1 schedules from being created during CHASM scheduler migration. | `dummy/workflow.go` |
| DeleteNamespace | `temporal-sys-delete-namespace-workflow` | `default-worker-tq` | `temporal-system` | Orchestrates the complete deletion of a namespace by marking it as deleted, renaming it, and asynchronously reclaiming all its workflow resources. | `deletenamespace/workflow.go` |
| DeleteNamespace | `temporal-sys-delete-executions-workflow` | `temporal-sys-delete-namespace-activity-tq` | `temporal-system` | Scans visibility storage and deletes workflow executions from a namespace in concurrent batches, using pagination and continue-as-new to handle large execution counts without history explosion. | `deletenamespace/deleteexecutions/workflow.go` |
| DeleteNamespace | `temporal-sys-reclaim-namespace-resources-workflow` | `temporal-sys-delete-namespace-activity-tq` | `temporal-system` | Ensures no new executions are created in a deleted namespace, waits for execution drainage, then removes the namespace from the database. | `deletenamespace/reclaimresources/workflow.go` |
| Scheduler | `temporal-sys-scheduler-workflow` | `temporal-sys-per-ns-tq` | per-namespace | Manages scheduled workflow execution by computing next scheduled times, handling overlapping executions, processing backfills, and maintaining schedule state with support for pause/resume and configuration updates. | `scheduler/fx.go` |
| Batcher | `temporal-sys-batch-workflow` | `temporal-sys-per-ns-tq` | per-namespace | Processes batch operations (terminate, cancel, signal, delete, reset, update options, unpause/reset activities) on workflow executions matched by a visibility query or explicit execution list. | `batcher/fx.go` |
| Batcher | `temporal-sys-batch-workflow-protobuf` | `temporal-sys-per-ns-tq` | per-namespace | Same as `temporal-sys-batch-workflow` but uses protobuf encoding for batch operation payloads. | `batcher/fx.go` |
| AddSearchAttributes | `temporal-sys-add-search-attributes-workflow` | `default-worker-tq` | `temporal-system` | Adds custom search attributes to the cluster by updating Elasticsearch mappings, waiting for cluster readiness, and persisting the new attributes to cluster metadata. | `addsearchattributes/workflow.go` |
| DLQ | `temporal-sys-dlq-workflow` | `default-worker-tq` | `temporal-system` | Handles dead letter queue operations: delete mode purges DLQ tasks up to a max message ID; merge mode re-enqueues DLQ tasks back to their source queues in batches. | `dlq/workflow.go` |
| ParentClosePolicy | `temporal-sys-parent-close-policy-workflow` | `temporal-sys-processor-parent-close-policy` | `temporal-system` | Processes parent close policy actions (abandon, terminate, or cancel) on child workflows when their parent closes, including cross-cluster signaling for remote executions. | `parentclosepolicy/workflow.go` |
| Scanner | `temporal-sys-tq-scanner-workflow` | `temporal-sys-tq-scanner-taskqueue-0` | `temporal-system` | Runs the task queue scavenger as a long-running background daemon to detect and clean up stale task queue data. | `scanner/workflow.go` |
| Scanner | `temporal-sys-history-scanner-workflow` | `temporal-sys-history-scanner-taskqueue-0` | `temporal-system` | Runs the history scavenger as a long-running background daemon to clean up orphaned workflow execution history data. | `scanner/workflow.go` |
| Scanner | `temporal-sys-executions-scanner-workflow` | `temporal-sys-executions-scanner-taskqueue-0` | `temporal-system` | Runs the executions scavenger as a long-running background daemon to detect and clean up dangling workflow executions across all shards. | `scanner/workflow.go` |
| Scanner (BuildID) | `build-id-scavenger` | `temporal-sys-build-id-scavenger-taskqueue-0` | `temporal-system` | Scans all task queue user data entries across namespaces and removes unused build IDs that haven't been the queue default for a configured duration. | `scanner/build_ids/scavenger.go` |
| WorkerDeployment | `temporal-sys-worker-deployment-workflow` | `temporal-sys-per-ns-tq` | per-namespace | Manages the complete worker deployment lifecycle including version creation/deletion, setting current/ramping versions, and enforcing task queue constraints. | `workerdeployment/util.go` |
| WorkerDeployment | `temporal-sys-worker-deployment-version-workflow` | `temporal-sys-per-ns-tq` | per-namespace | Manages a specific worker deployment version lifecycle including task queue registration, metadata and compute config updates, drainage state, and deletion. | `workerdeployment/util.go` |
| Migration | `catchup` | `default-worker-tq` | `temporal-system` | Waits for a specified cluster to catch up on replication lag to the current state before proceeding with migration operations. | `migration/catchup_workflow.go` |
| Migration | `force-replication` | `default-worker-tq` | `temporal-system` | Force-replicates workflow executions from one cluster to another by listing executions and enqueuing replication tasks, with optional verification on the target cluster. | `migration/force_replication_workflow.go` |
| Migration | `force-replication-v2` | `default-worker-tq` | `temporal-system` | V2 of force replication with improved batching and verification of successful replication on the target cluster. | `migration/force_replication_workflow.go` |
| Migration | `namespace-handover` | `default-worker-tq` | `temporal-system` | Orchestrates namespace failover from one cluster to another by waiting for replication catch-up, putting the namespace in handover state, draining replication tasks, and updating the active cluster. | `migration/handover_workflow.go` |
| Migration | `namespace-handover-v2` | `default-worker-tq` | `temporal-system` | V2 of namespace handover with an improved handover protocol and replication drain sequencing. | `migration/handover_workflow.go` |
| Migration | `force-task-queue-user-data-replication` | `default-worker-tq` | `temporal-system` | Replicates task queue user data entries (worker versioning info) from source to target cluster with pagination and heartbeat tracking. | `migration/force_replication_workflow.go` |

> **Note:** Workflows with namespace `per-namespace` are registered as `PerNSWorkerComponent` and run in every namespace using the `temporal-sys-per-ns-tq` task queue. Workflows with namespace `temporal-system` are system-level and run only in the `temporal-system` namespace.

---

## Activity Types

### DeleteNamespace (`deletenamespace/activities.go`)
- `GetNamespaceInfoActivity`
- `ValidateProtectedNamespacesActivity`
- `ValidateNexusEndpointsActivity`
- `MarkNamespaceDeletedActivity`
- `GenerateDeletedNamespaceNameActivity`
- `RenameNamespaceActivity`

### DeleteNamespace / deleteexecutions
- `GetNextPageTokenActivity`
- `DeleteExecutionsActivity`

### DeleteNamespace / reclaimresources
- `IsAdvancedVisibilityActivity`
- `CountExecutionsAdvVisibilityActivity`
- `EnsureNoExecutionsAdvVisibilityActivity`
- `DeleteNamespaceActivity`
- `GetNamespaceCacheRefreshInterval`

### Batcher (`batcher/activities.go`)
- `BatchActivityWithProtobuf`

### AddSearchAttributes (`addsearchattributes/workflow.go`)
- `AddESMappingFieldActivity`
- `WaitForYellowStatusActivity`
- `UpdateClusterMetadataActivity`

### DLQ (`dlq/workflow.go`)
- `dlq-delete-tasks-activity`
- `dlq-read-tasks-activity`
- `dlq-re-enqueue-tasks-activity`

### ParentClosePolicy (`parentclosepolicy/workflow.go`)
- `temporal-sys-parent-close-policy-activity`

### Scanner (`scanner/workflow.go`)
- `temporal-sys-tq-scanner-scvg-activity`
- `temporal-sys-history-scanner-scvg-activity`
- `temporal-sys-executions-scanner-scvg-activity`

### Scanner / BuildID (`scanner/build_ids/scavenger.go`)
- `scavenge-build-ids`

### WorkerDeployment / version (`workerdeployment/version_activities.go`)
- `StartWorkerDeploymentWorkflow`
- `SyncDeploymentVersionUserData`
- `CheckIfTaskQueuesHavePollers`
- `UpdateWorkerControllerInstance`
- `DeleteWorkerControllerInstance`

### WorkerDeployment / main (`workerdeployment/activities.go`)
- `SyncWorkerDeploymentVersion`
- `DeleteWorkerDeploymentVersion`
- `DescribeVersionFromWorkerDeployment`
- `StartWorkerDeploymentVersionWorkflow`
- `UpdateWorkerControllerInstanceFromDeployment`

---

## Dynamic Configs

Configs listed per workflow using their JSON key names. Workflows with no entry use no dynamic config directly.

### `temporal-sys-delete-namespace-workflow` / `temporal-sys-delete-executions-workflow` / `temporal-sys-reclaim-namespace-resources-workflow`
- `worker.protectedNamespaces`
- `worker.deleteNamespaceActivityLimitsConfig`
- `frontend.deleteNamespaceDeleteActivityRPS`
- `frontend.allowDeleteNamespaceIfNexusEndpointTarget`
- `nexus.endpointListDefaultPageSize`
- `namespace.cacheRefreshInterval`

### `temporal-sys-scheduler-workflow`
- `worker.enableScheduler`
- `worker.enableCHASMSchedulerMigration`
- `worker.schedulerNamespaceStartWorkflowRPS`
- `worker.schedulerLocalActivitySleepLimit`
- `limit.blobSize.error`

### `temporal-sys-batch-workflow` / `temporal-sys-batch-workflow-protobuf`
- `worker.enableNamespaceBatcher`
- `worker.batcherRPS`
- `worker.batcherConcurrency`

### `temporal-sys-tq-scanner-workflow`
- `worker.taskQueueScannerEnabled`
- `worker.scannerPersistenceMaxQPS`
- `worker.ScannerMaxConcurrentActivityExecutionSize`
- `worker.ScannerMaxConcurrentWorkflowTaskExecutionSize`
- `worker.ScannerMaxConcurrentActivityTaskPollers`
- `worker.ScannerMaxConcurrentWorkflowTaskPollers`

### `temporal-sys-history-scanner-workflow`
- `worker.historyScannerEnabled`
- `worker.historyScannerDataMinAge`
- `worker.historyScannerVerifyRetention`

### `temporal-sys-executions-scanner-workflow`
- `worker.executionsScannerEnabled`
- `worker.executionScannerPerHostQPS`
- `worker.executionScannerPerShardQPS`
- `worker.executionDataDurationBuffer`
- `worker.executionScannerWorkerCount`
- `worker.executionEnableHistoryEventIdValidator`

### `build-id-scavenger`
- `worker.buildIdScavengerEnabled`
- `worker.removableBuildIdDurationSinceDefault`
- `worker.buildIdScavengerVisibilityRPS`

### `temporal-sys-worker-deployment-workflow` / `temporal-sys-worker-deployment-version-workflow`
- `matching.maxDeployments`
- `matching.maxTaskQueuesInDeploymentVersion`
- `matching.maxVersionsInDeployment`
- `matching.deploymentWorkflowVersion`
- `matching.versionDrainageStatusRefreshInterval`
- `matching.versionDrainageStatusVisibilityGracePeriod`
- `worker.reactivationSignalDedupCacheMaxSize`
- `frontend.visibilityMaxPageSize`
- `limit.maxIDLength`

### `catchup` / `force-replication` / `force-replication-v2` / `namespace-handover` / `namespace-handover-v2` / `force-task-queue-user-data-replication`
- `worker.generateMigrationTaskViaFrontend`
- `worker.enableHistoryRateLimiter`
