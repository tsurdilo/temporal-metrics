# Workflow Task Timeout Alerts

Standalone alert set for customers running workloads sensitive to workflow task timeouts, particularly those using heavy local-activity patterns.

Three alerts covering distinct WFT timeout failure modes:

| Alert | Metric | Severity | Fires when |
|-------|--------|----------|-----------|
| WFT-1: WFT StartToClose Timeout | `start_to_close_timeout{operation="TimerActiveTaskWorkflowTaskTimeout"}` | CRITICAL | A single WFT exceeded its `workflowTaskTimeout` execution deadline |
| WFT-2: WFT Heartbeat Timeout | `workflow_task_heartbeat_timeout_count` | CRITICAL | A WFT has been heartbeating with no forward progress for longer than `WorkflowTaskHeartbeatTimeout` (default 30 min) |
| WFT-3: WFT ScheduleToStart Timeout | `schedule_to_start_timeout{operation="TimerActiveTaskWorkflowTaskTimeout"}` | WARNING | A WFT on the sticky task queue was not picked up within `StickyScheduleToStartTimeout` (default 5s) |

All are server-side metrics emitted by the History service — they fire regardless of which Temporal SDK the customer uses.

WFT-1, WFT-2, and WFT-3 represent independent failure modes and will not necessarily fire together.

---

## Prerequisites

- Temporal Server v1.20+ emitting Prometheus metrics to Grafana
- Grafana 9.0+ with unified alerting enabled

---

## Method 1 — Provisioning file (file system access required)

Use this if you can write files to the Grafana server host.

**Step 1 — look up your Prometheus datasource UID:**

```bash
curl -s http://admin:admin@localhost:3000/api/datasources \
  | jq '.[] | select(.type=="prometheus") | {name, uid}'
```

**Step 2 — substitute the UID and copy into Grafana provisioning:**

```bash
GRAFANA_DS_UID="<your-uid-here>"

sed "s/datasourceUid: Prometheus/datasourceUid: ${GRAFANA_DS_UID}/g" \
  temporal-wft-timeout-alerts.yaml \
  > /path/to/grafana/provisioning/alerting/temporal-wft-timeout-alerts.yaml
```

**Step 3 — reload Grafana:**

Restart Grafana or wait for the provisioning hot-reload interval. Alerts appear under **Alerting → Alert rules → Temporal Server → temporal-wft-timeout**.

---

## Method 2 — Grafana HTTP API (no file system access required)

Use this with Grafana Cloud or any managed Grafana instance where you have HTTP access but not file system access.

**Step 1 — look up your Prometheus datasource UID:**

```bash
curl -s -u admin:admin http://localhost:3000/api/datasources \
  | jq '.[] | select(.type=="prometheus") | {name, uid}'
```

**Step 2 — look up or create the alert folder UID:**

```bash
# List existing folders
curl -s -u admin:admin http://localhost:3000/api/folders \
  | jq '.[] | {title, uid}'

# Create the folder if it does not exist
curl -s -u admin:admin -X POST http://localhost:3000/api/folders \
  -H "Content-Type: application/json" \
  -d '{"title": "Temporal Server"}' \
  | jq '{title, uid}'
```

**Step 3 — substitute the datasource UID into the JSON file:**

```bash
GRAFANA_DS_UID="<your-datasource-uid>"

sed "s/REPLACE_WITH_DATASOURCE_UID/${GRAFANA_DS_UID}/g" \
  temporal-wft-timeout-alerts-api.json \
  > temporal-wft-timeout-alerts-api-ready.json
```

**Step 4 — PUT the rule group:**

```bash
GRAFANA_FOLDER_UID="<your-folder-uid>"
GRAFANA_HOST="http://localhost:3000"
GRAFANA_USER="admin:admin"

curl -s -u "${GRAFANA_USER}" \
  -X PUT "${GRAFANA_HOST}/api/v1/provisioning/folder/${GRAFANA_FOLDER_UID}/rule-groups/temporal-wft-timeout" \
  -H "Content-Type: application/json" \
  -d @temporal-wft-timeout-alerts-api-ready.json
```

A `200 OK` response means the rule group was created or updated. Alerts appear immediately under **Alerting → Alert rules → Temporal Server → temporal-wft-timeout**.

**To verify:**

```bash
curl -s -u "${GRAFANA_USER}" \
  "${GRAFANA_HOST}/api/v1/provisioning/folder/${GRAFANA_FOLDER_UID}/rule-groups/temporal-wft-timeout" \
  | jq '.rules[].title'
```

---

## Notification policies

Wire alerts into your routing in **Alerting → Notification policies** using these labels:

```
severity: critical   # WFT-1 and WFT-2
severity: warning    # WFT-3
service: temporal
component: history
```

---

## Tuning

### Namespace scope

All queries use `namespace=~".+"` to match all namespaces. To scope to a specific namespace:

```promql
namespace="your-namespace-name"
```

Edit the `expr` field in the YAML or JSON before deploying.

### `for` durations

| Alert | Default | Notes |
|-------|---------|-------|
| WFT-1 StartToClose timeout | 1m | Short — each event is a concrete WFT execution failure |
| WFT-2 Heartbeat timeout | 1m | Short — 30-minute threshold already baked in; any rate is significant |
| WFT-3 ScheduleToStart timeout | 5m | Longer — warning-level, allow for transient sticky queue misses |

---

## Alert details

### WFT-1 — Workflow Task StartToClose Timeout

**Metric:** `start_to_close_timeout{operation="TimerActiveTaskWorkflowTaskTimeout"}`
**Emitted by:** History service, timer queue active task executor
**Severity:** CRITICAL | **For:** 1m

Fires when the server received no workflow task completion or failure response within the `workflowTaskTimeout` deadline (default 10s).

For workflow tasks running local activities, this can indicate:

- **gRPC message size exceeded** — total results of all local activities within the workflow task exceeded the default gRPC message size limit (4MB), preventing the response from being sent.
- **Heartbeat not received** — the workflow task heartbeat sent by the SDK at 80% of the WFT timeout was not received by the server, or was throttled, before the timeout fired.

In both cases, local activities will run again on workflow task retry, potentially on a different worker.

**What to check when this alert fires:**

1. **Server dashboard — Resource Exhausted graphs:** check whether `ResourceExhausted` errors are appearing on the `RespondWorkflowTaskCompleted` operation. This indicates the server is throttling the WFT response.

2. **SDK dashboard — `temporal_request_failure` metric:** filter for `operation=RespondWorkflowTaskCompleted` and check the `status_code` label.
   - If `RESOURCE_EXHAUSTED`: determine whether this is server-side throttling (RPS/QPS limits) or gRPC message size rejection. If you have a gRPC proxy in front of Temporal, check whether the proxy is the one returning `RESOURCE_EXHAUSTED` due to message size limits.

3. **Server persistence latencies — `UpdateWorkflowExecution`:** high DB latency on this operation can cause the WFT timer to fire before the WFT heartbeat response completes, even if the worker sent the heartbeat in time.

4. **SDK metric `temporal_worker_task_slots_available` for `worker_type=LocalActivityWorker`:** if available slots are at 0, local activities may be hanging — all slots occupied by already-timed-out local activities that are not releasing. In this case, restarting the affected workers is the most effective remediation.

---

### WFT-2 — Workflow Task Heartbeat Timeout

**Metric:** `workflow_task_heartbeat_timeout_count`
**Emitted by:** History service, `RespondWorkflowTaskCompleted` path
**Severity:** CRITICAL | **For:** 1m

Fires when local activity or activities, including all their retries, ran for more than 30 minutes without completing or failing (`WorkflowTaskHeartbeatTimeout`, default 30 minutes, configurable via `history.workflowTaskHeartbeatTimeout`).

Local activities should always have a `ScheduleToClose` timeout set to less than 30 minutes — the `WorkflowTaskHeartbeatTimeout` is the absolute ceiling. In practice it should be set much lower.

---

### WFT-3 — Workflow Task ScheduleToStart Timeout

**Metric:** `schedule_to_start_timeout{operation="TimerActiveTaskWorkflowTaskTimeout"}`
**Emitted by:** History service, timer queue active task executor
**Severity:** WARNING | **For:** 5m

Fires when a workflow task scheduled on the sticky task queue was not picked up by the owning worker within the sticky schedule-to-start timeout (default 5s, set by the worker SDK via `StickyScheduleToStartTimeout`).

This timeout only applies to sticky task queue workflow tasks — normal task queue workflow tasks do not have a ScheduleToStart timeout. When this fires, the server reschedules the workflow task on the normal task queue.
