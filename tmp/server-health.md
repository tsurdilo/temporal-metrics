# Temporal History Host Health — Monitoring Runbook

> **Source basis**: All information in this runbook is derived directly from the Temporal server source code (`service/history/deep_health_check.go`, `service/history/service.go`, `service/history/shard/controller_impl.go`, `common/dynamicconfig/constants.go`, `service/frontend/health_check.go`) as reviewed in the [temporalio/temporal](https://github.com/temporalio/temporal) repository. Anything not confirmed from code is explicitly marked.

---

## 1. What `host_health` actually is

The `host_health` gauge (`HistoryHostHealthGauge`) is emitted by each history pod at the end of `DeepHealthCheck`. It is a numeric value representing that pod's health state at the time the check ran.

> **⚠️ This metric is absent by default.** `host_health` is only emitted when `DeepHealthCheck` is explicitly called on each history pod. There is no internal background polling loop in the Temporal server that drives this. If nothing in your infrastructure is calling `AdminHandler.DeepHealthCheck` on the frontend, this metric will not exist in your metrics store and all queries and alerts based on it will produce no data. See section 9 for how to set up the required poller.

### Health state values (from `enumsspb.HEALTH_STATE_*`)

| Value | State | Meaning |
|---|---|---|
| `0` | `UNSPECIFIED` | Unknown / pod not reporting — treat as missing data |
| `1` | `SERVING` | All checks passed — pod is healthy and ready |
| `2` | `NOT_SERVING` | RPC or persistence threshold exceeded — pod is degraded |
| `3` | `DECLINED_SERVING` | gRPC health is not `SERVING` — covers startup incomplete, shutdown, or crashloopbackoff |

**Important**: the per-pod `host_health` gauge **never emits value `3`**. It only records `0` (UNSPECIFIED — metric absent), `1` (SERVING), or `2` (NOT_SERVING). Value `3` (DECLINED_SERVING) only appears as a fleet-level aggregate state returned by the frontend's `healthCheckerImpl.Check` — it is never written to the per-pod gauge. See section 2 for the frontend aggregation logic that produces `DECLINED_SERVING` at the fleet level.

### The 5 checks run inside `DeepHealthCheck` (from `deep_health_check.go`)

Checks run sequentially and **short-circuit on first failure**:

| Order | Check | Source | Failure threshold | Fails to state |
|---|---|---|---|---|
| 1 | gRPC health | `healthServer.Check()` | Error calling Check, or non-SERVING status after the 60s init window (see note below) | `NOT_SERVING` (2) |
| 2 | RPC average latency | `historyHealthSignal.AverageLatency()` | `> HealthRPCLatencyFailure` (default **500ms**) | `NOT_SERVING` (2) |
| 3 | RPC error ratio | `historyHealthSignal.ErrorRatio()` | `> HealthRPCErrorRatio` (default **0.90 = 90%**) | `NOT_SERVING` (2) |
| 4 | Persistence average latency | `persistenceHealthSignal.AverageLatency()` | `> HealthPersistenceLatencyFailure` (default **500ms**) | `NOT_SERVING` (2) |
| 5 | Persistence error ratio | `persistenceHealthSignal.ErrorRatio()` | `> HealthPersistenceErrorRatio` (default **0.90 = 90%**) | `NOT_SERVING` (2) |

**Note on Check 1 — startup suppression (`suppressStartupErrors`)**: for the first `history.healthHistoryInitializationTime` (default **60 seconds**) after a pod starts, a non-SERVING gRPC health result from Check 1 is suppressed and treated as SERVING. This prevents false `NOT_SERVING` readings during normal shard acquisition. After the 60s window, a non-SERVING gRPC health result produces `host_health = 2`. This 60s window is separate from the 5-second stabilization delay in section 1b.

**Important**: the default thresholds are deliberately conservative — 90% error ratio and 500ms average latency. These are designed to catch severe degradation, not transient spikes. If a pod flips to `NOT_SERVING` under these defaults, something is genuinely wrong.

### When is `DeepHealthCheck` invoked?

Confirmed from source code (`workflow_handler.go`, `health.go` interceptor):

**`DeepHealthCheck` has no internal background polling loop.** It is an on-demand RPC only. The `host_health` metric is emitted only when something explicitly calls `DeepHealthCheck` on a history pod.

The `HealthInterceptor` — which guards every frontend `WorkflowService` and `OperatorService` request — is completely separate. From `health.go` and `workflow_handler.go`:

```go
// Called once on frontend startup (after membership ready):
wh.healthInterceptor.SetHealthy(true)

// Called once on frontend shutdown:
wh.healthInterceptor.SetHealthy(false)
```

This interceptor has nothing to do with history health or `host_health`. It only controls whether the frontend itself accepts requests.

**`healthCheckerImpl.Check`** — the fan-out to all history hosts — is only wired to `AdminHandler.DeepHealthCheck`. It runs when something calls that admin API endpoint explicitly.

**Practical implication**: if `host_health` data is present in your metrics, something in your environment is calling `AdminHandler.DeepHealthCheck` on a schedule (monitoring infrastructure, Kubernetes probes, or similar). If nothing calls it, the metric is absent or stale. There is no Temporal server component that polls this internally on a timer — confirm with your infrastructure what is driving the calls and at what interval, as this determines how fresh the metric is for alerting.

---

## 1b. Startup and shutdown health transitions (from `service.go`, `controller_impl.go`)

This section explains what controls the gRPC health state that check 1 reads — and therefore what causes `NOT_SERVING` (value 2) during startup and shutdown.

### The asymmetric health transition pattern

The history service deliberately uses different rules for becoming healthy vs unhealthy:

- **Becoming unhealthy**: immediate — any failure triggers it instantly
- **Becoming healthy**: requires sustained proof — multiple conditions must be met plus a stabilization delay

This prevents a pod from briefly passing a health check before it has fully initialized, which would cause it to receive traffic it cannot handle.

### Startup sequence (from `service.go`)

```
Pod starts → gRPC health = NOT_SERVING
    ↓
DeepHealthCheck called within first 60s (HealthHistoryInitializationTime):
    suppressStartupErrors suppresses gRPC NOT_SERVING → host_health = 1
    ↓
After 60s init window expires: gRPC NOT_SERVING → host_health = 2
    ↓
InitialShardsAcquired completes (all 3 conditions below)
    ↓
5 second stabilization delay
    ↓
gRPC health = SERVING → host_health = 1
```

From `service.go`:
```go
s.healthServer.SetServingStatus(serviceName, healthpb.HealthCheckResponse_NOT_SERVING)
go func() {
    if s.handler.controller.InitialShardsAcquired(readinessCtx) == nil {
        if util.InterruptibleSleep(readinessCtx, 5*time.Second) == nil {
            s.healthServer.SetServingStatus(serviceName, healthpb.HealthCheckResponse_SERVING)
        }
    }
}()
```

### `InitialShardsAcquired` — the 3 conditions that must all pass (from `controller_impl.go`)

Before the 5-second stabilization timer even starts, all three of these must be true:

1. **Joined membership ring** — pod must own at least one shard
2. **Acquired all expected shards** — pod has ownership of all shards it is supposed to own
3. **Shard readiness check passes for each shard** — `GetShardByID`, `GetEngine`, and `AssertOwnership` all succeed per shard

For a large cluster (e.g. 2048 shards across 14 history pods, ~146 shards per pod), acquiring all expected shards during startup or after a membership change takes non-trivial time. During the first 60s (`HealthHistoryInitializationTime`), `suppressStartupErrors` masks this — `host_health` shows `1`. After the 60s window, a pod still in shard acquisition shows `host_health = 2` until acquisition completes and gRPC flips to SERVING.

### Shutdown sequence

```go
s.healthServer.SetServingStatus(serviceName, healthpb.HealthCheckResponse_NOT_SERVING)
```

Shutdown is immediate — no conditions, no delay. Pod flips to `NOT_SERVING` gRPC state instantly. Once the 60s `suppressStartupErrors` init window has long since passed on a running pod, any call to `DeepHealthCheck` during or after shutdown will see Check 1 fail and record `host_health = 2` immediately.

### Practical implications for alerting

| Scenario | `host_health` value | Expected? |
|---|---|---|
| Pod starting up, shard acquisition within 60s init window | `1` | Yes — gRPC NOT_SERVING suppressed by `suppressStartupErrors` |
| Pod starting up, shard acquisition past 60s init window | `2` | Investigate — acquisition taking unusually long |
| Pod fully started, all shards acquired + 5s passed | `1` | Yes — healthy |
| Pod shutting down gracefully | `2` | Yes — normal during rolling restarts |
| Pod in crashloopbackoff (past 60s init window) | `2` | Abnormal — pod never reaches `SERVING` |
| Pod healthy but persistence degraded | `2` | Abnormal — investigate DB |
| Pod healthy but RPC latency/errors high | `2` | Abnormal — investigate traffic or shard contention |

**Distinguishing crashloop from normal startup**: a pod that stays at `host_health = 2` for longer than expected (beyond a few minutes for a normal startup, once past the 60s init window) is likely crashlooping or stuck in shard acquisition. Check pod restart count in Kubernetes alongside the `host_health` timeline.

---

## 2. Dynamic config thresholds (all tunable at runtime without restart)

From `common/dynamicconfig/constants.go`:

### Per-host thresholds (control when a single pod flips to NOT_SERVING)

| Dynamic config key | Default | Description |
|---|---|---|
| `history.healthRPCLatencyFailure` | `500` (ms) | RPC average latency threshold |
| `history.healthRPCErrorRatio` | `0.90` | RPC error ratio threshold |
| `history.healthPersistenceLatencyFailure` | `500` (ms) | Persistence average latency threshold |
| `history.healthPersistenceErrorRatio` | `0.90` | Persistence error ratio threshold |

### Aggregation thresholds (frontend-level, from `healthCheckerImpl.Check`)

These are evaluated by the frontend's `healthCheckerImpl` which fans out `DeepHealthCheck` calls to all history hosts in parallel and aggregates the results. Confirmed from source code.

**Check 1 — DECLINED_SERVING proportion (runs first):**

```
hostDeclinedServingProportion = DECLINED_SERVING_count / total_hosts
if hostDeclinedServingProportion > hostDeclinedServingProportion_threshold:
    → frontend returns DECLINED_SERVING
    → logs: "health check exceeded host declined serving proportion threshold"
```

| Dynamic config key | Default | Notes |
|---|---|---|
| `frontend.historyHostSelfErrorProportion` | `0.05` (5%) | Minimum of 2 hosts must be `DECLINED_SERVING` regardless of proportion — enforced by `ensureMinimumProportionOfHosts`. For a 14-pod fleet, effective minimum threshold is `2/14 = 14.3%` |

**Check 2 — Combined failure proportion (runs second):**

```
failedHostProportion = (NOT_SERVING_count + UNSPECIFIED_count + INTERNAL_ERROR_count + other_count) / total_hosts
combinedProportion = failedHostProportion + DECLINED_SERVING_proportion
if combinedProportion > hostFailurePercentage_threshold:
    → frontend returns NOT_SERVING
    → logs: "health check exceeded host failure percentage threshold"
```

| Dynamic config key | Default | Notes |
|---|---|---|
| `frontend.historyHostErrorPercentage` | `0.50` (50%) | Counts all non-SERVING, non-DECLINED_SERVING pods (`NOT_SERVING`, `UNSPECIFIED`, `INTERNAL_ERROR`, other) plus `DECLINED_SERVING` pods combined. Unreachable pods are set to `NOT_SERVING` and counted in the non-SERVING bucket |

**Important nuances from the code:**
- If the membership resolver returns zero history hosts, the frontend immediately returns `NOT_SERVING` — this happens before any pings are attempted and produces no log message, just the state
- `UNSPECIFIED` (value 0) counts as a failure — it falls into the `default:` case of the aggregation switch alongside `NOT_SERVING` and `INTERNAL_ERROR`
- `failed to ping deep health check` log → that pod's response is explicitly set to `NOT_SERVING` (not UNSPECIFIED) → counts toward `failedHostCount`
- The combined 50% check includes pods in `DECLINED_SERVING` — so a cluster doing a large rolling restart where many pods are simultaneously draining can trip the `NOT_SERVING` threshold even if no individual pod has a health problem

**For a 14-pod history fleet:**

| Scenario | Pods in state | Proportion | Threshold crossed? |
|---|---|---|---|
| 1 pod `DECLINED_SERVING` | 1/14 = 7.1% | Below minimum-2 rule | No |
| 2 pods `DECLINED_SERVING` | 2/14 = 14.3% | > 5% AND ≥ 2 pods | Yes → frontend `DECLINED_SERVING` |
| 7 pods any unhealthy state | 7/14 = 50.0% | = 50%, not > 50% | No — condition is strictly `>` |
| 8 pods any unhealthy state | 8/14 = 57.1% | > 50% | Yes → frontend `NOT_SERVING` |

To tighten thresholds (catch degradation earlier), update dynamic config:

```yaml
history.healthPersistenceLatencyFailure:
  - value: 200   # ms — tighter than 500ms default

history.healthPersistenceErrorRatio:
  - value: 0.50  # 50% — tighter than 90% default

history.healthRPCLatencyFailure:
  - value: 200

history.healthRPCErrorRatio:
  - value: 0.50
```

Tradeoff: tighter thresholds mean earlier signal but higher sensitivity to transient spikes.

---

## 3. Core Grafana queries

### Query 1 — Absolute pod count by health state

Use this as the primary dashboard panel. Shows how many history pods are in each state at any point in time.

```promql
sum by (health_state) (
  label_replace(
    clamp_max(host_health{cluster="$cluster", service_name="history"} == 1, 1),
    "health_state", "serving", "", ""
  )
) or
sum by (health_state) (
  label_replace(
    clamp_max(host_health{cluster="$cluster", service_name="history"} == 2, 1),
    "health_state", "not_serving", "", ""
  )
)
```

**How it works**: `host_health == N` filters to only time series where the gauge equals that value. `clamp_max(..., 1)` normalizes each series to 1 so `sum` counts pods rather than summing the gauge values. `label_replace` adds a human-readable `health_state` label. The two groups are unioned with `or`. There is no `== 3` group — the per-pod gauge never emits DECLINED_SERVING.

### Query 2 — Percentage of pods by health state

Use this alongside Query 1. Useful for alerting thresholds that should scale with fleet size.

```promql
(
  sum by (health_state) (
    label_replace(
      clamp_max(host_health{cluster="$cluster", service_name="history"} == 1, 1),
      "health_state", "serving", "", ""
    )
  ) or
  sum by (health_state) (
    label_replace(
      clamp_max(host_health{cluster="$cluster", service_name="history"} == 2, 1),
      "health_state", "not_serving", "", ""
    )
  )
) / scalar(
  count(host_health{cluster="$cluster", service_name="history"})
) * 100
```

**How it works**: same numerator as Query 1. `scalar(count(host_health{...}))` extracts the total number of history pods reporting the metric as a single number. Dividing each state count by total pod count and multiplying by 100 gives percentage. `scalar()` is required here so the single total count value can be used as a divisor across the labeled groups.

---

## 4. Suggested alert thresholds

For a 14-pod history fleet (adjust `count` for your fleet size):

| Alert | PromQL condition | Severity | Meaning |
|---|---|---|---|
| Any pod degraded | `not_serving count >= 1` | Warning | At least one pod has crossed a health threshold |
| Multiple pods degraded | `not_serving count >= 3` | Critical | Systemic issue — investigate DB or RPC signals immediately |
| Significant draining | `declined_serving count >= 3` simultaneously | Warning | More pods draining than expected — may indicate bad deploy or cascading shard loss |
| Majority unhealthy | `(not_serving + declined_serving) / total > 50%` | Critical | Cluster severely degraded — consider failover |

---

## 5. Diagnosing what triggered NOT_SERVING

`host_health == 2` tells you a pod is degraded but not which of the 5 checks fired. Start with logs, then cross-reference metrics.

### Step 1 — Check frontend logs for these messages

These log messages are emitted by the frontend when it detects history host health problems:

| Log message | Meaning |
|---|---|
| `failed to ping deep health check` | Frontend goroutine could not reach a history host. The pod's response is explicitly set to `NOT_SERVING` (2) — counts toward `failedHostCount` in the combined failure proportion check |
| `health check exceeded host declined serving proportion threshold` | `DECLINED_SERVING` pod count / total pods exceeded `frontend.historyHostSelfErrorProportion` (default 5%, minimum 2 pods) |
| `health check exceeded host failure percentage threshold` | Combined (`NOT_SERVING` + `UNSPECIFIED` + `DECLINED_SERVING`) / total pods exceeded `frontend.historyHostErrorPercentage` (default 50%) |

The `failed to ping` log message will often appear alongside the threshold breach messages — each unreachable pod contributes to the 50% combined threshold.

### Step 2 — Cross-reference metrics

### Was it persistence (checks 4–5)?

Look for spikes in persistence latency and error rates at the same timestamp as the `host_health` flip:

```promql
# Persistence latency p99
histogram_quantile(0.99,
  rate(persistence_latency_bucket{
    cluster="$cluster",
    service_name="history"
  }[5m])
)
```

```promql
# Persistence error rate
rate(persistence_requests{
  cluster="$cluster",
  service_name="history",
  result="failed"
}[5m])
```

If these spiked at the same time as `host_health` went to 2, persistence is the trigger. Investigate your database and connection layer for errors at the same timestamp.

### Was it RPC latency/errors (checks 2–3)?

```promql
# History RPC latency p99
histogram_quantile(0.99,
  rate(service_latency_bucket{
    cluster="$cluster",
    service_name="history"
  }[5m])
)
```

If history RPC latency spiked without DB issues — for example due to shard acquisition contention, workflow lock queuing, or membership changes — that's the RPC check firing.

### Was it gRPC health not SERVING — check 1 firing (host_health = 2 with no persistence/RPC spike)?

If `host_health = 2` without any corresponding persistence or RPC metric spike, the likely cause is Check 1 failing — the gRPC health server is not in `SERVING` state. This covers three distinct situations:

**1. Normal startup past the 60s init window** — pod is still acquiring shards. During the first 60s (`HealthHistoryInitializationTime`), `suppressStartupErrors` masks this and the pod shows `host_health = 1`. After 60s, a non-SERVING gRPC result is no longer suppressed and the pod shows `host_health = 2`. Duration depends on shard count (~146 shards per pod at 2048 total / 14 pods).

**2. Normal shutdown / rolling restart** — pod received shutdown signal, immediately set gRPC health to NOT\_SERVING. Shows `host_health = 2`. Expected during deploys.

**3. Crashloopbackoff** — pod is restarting repeatedly and never completes `InitialShardsAcquired`. Shows `host_health = 2` after the 60s init window on each restart cycle.

To distinguish between these, cross-reference:
- **Kubernetes pod restart count** — if count is increasing, it's a crashloop
- **Deploy/restart timing** — if a rolling restart is in progress, `host_health = 2` on some pods is expected
- **Duration** — a pod stuck at `2` for more than a few minutes outside of a deploy window needs investigation
- **Pod logs** — shard acquisition errors or panic messages will appear during crashloop

---

## 6. Service health checks — general overview

All three Temporal services expose the standard gRPC health protocol (`grpc.health.v1.Health/Check`). You can probe them with `grpc-health-probe`:

```bash
# Frontend
grpc-health-probe -addr=<host:7233> -service=temporal.api.workflowservice.v1.WorkflowService

# History
grpc-health-probe -addr=<host:7234> -service=temporal.api.workflowservice.v1.HistoryService

# Matching
grpc-health-probe -addr=<host:7235> -service=temporal.api.workflowservice.v1.MatchingService
```

### What each probe actually tells you (confirmed from source code)

| Service | Flips to SERVING when | Flips to NOT_SERVING when | What the probe tells you |
|---|---|---|---|
| **Frontend** | `membershipMonitor.WaitUntilInitialized()` completes | Shutdown | Pod is up and joined membership ring |
| **History** | `InitialShardsAcquired()` (3 conditions) + 5s stabilization | Shutdown (after drain wait) | Pod is up, owns all expected shards, engine ready |
| **Matching** | Immediately after `handler.Start()` — no conditions | Shutdown (after drain wait) | Pod is up and handler started — **no deeper readiness guarantee** |

The matching probe is notably weaker than history — matching flips to `SERVING` immediately on startup with no shard acquisition or stabilization requirement. A matching pod reporting `SERVING` via gRPC health does not mean it has loaded any task queues.

### Kubernetes readiness probes

If you are using the Temporal Helm chart, the frontend readiness probe is already configured out of the box:

```yaml
frontend:
  readinessProbe:
    grpc:
      port: 7233
      service: temporal.api.workflowservice.v1.WorkflowService
```

K8s pod readiness state is available in Prometheus via:

```promql
# Frontend pods ready
kube_pod_status_ready{
  namespace="$namespace",
  pod=~".*frontend.*",
  condition="true"
}

# History pods ready (gRPC health — shard acquisition complete)
kube_pod_status_ready{
  namespace="$namespace",
  pod=~".*history.*",
  condition="true"
}

# Matching pods ready
kube_pod_status_ready{
  namespace="$namespace",
  pod=~".*matching.*",
  condition="true"
}
```

These are general pod liveness signals — they tell you pods are up and past their startup conditions. They are **not** a substitute for `host_health` when making a failover decision about the history fleet.

---

## 7. Cluster failover

For a failover decision, the gRPC health probes above are necessary but not sufficient. A history pod can pass its gRPC readiness probe (shards acquired, `SERVING`) and still have degraded history health if persistence latency or RPC error ratios exceed thresholds. The `host_health` metric is the signal that captures this deeper degradation.

**Failover is a manual decision.** There is no automatic cross-cluster failover triggered by any of these health signals.

### The complete signal stack for a failover decision

| Layer | Signal | Meaning |
|---|---|---|
| 1 | Frontend `kube_pod_status_ready` | Frontend is reachable — prerequisite for everything else |
| 2 | `host_health` metric present and fresh | Poller is reaching frontend, `DeepHealthCheck` fan-out is working |
| 3 | `host_health == 2` on majority of history pods | History fleet is genuinely degraded beyond RPC/persistence thresholds |

If layer 1 fails (frontend pods down), layer 2 will also fail (no metric emission). Absence of `host_health` data combined with frontend pods not ready is itself a failover signal. You can detect metric staleness with:

```promql
# Alert if host_health hasn't been updated recently
time() - max(timestamp(host_health{cluster="$cluster", service_name="history"})) > 120
```

### When to consider initiating failover

- Frontend pods down AND not recovering within SLA
- `host_health` metric absent/stale AND frontend confirmed unreachable
- `host_health == 2` on > 50% of history pods AND confirmed infrastructure issue AND not recovering within SLA

---

## 9. Calling `DeepHealthCheck` to drive the `host_health` metric

Because `host_health` is only emitted when `AdminHandler.DeepHealthCheck` is called explicitly (no internal polling loop), you need something in your infrastructure calling it on a regular schedule for the metric to be useful.

A single call to `AdminHandler.DeepHealthCheck` on the **frontend** fans out to all history hosts via `healthCheckerImpl.Check` and returns an aggregated state. You do not need to call individual history pods.

The admin service is exposed on the same port as the frontend (default `7233`).

### Example Go poller

```go
package main

import (
	"context"
	"log"
	"os"
	"os/signal"
	"syscall"
	"time"

	adminservice "go.temporal.io/server/api/adminservice/v1"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

func main() {
	frontendAddr := "temporal-frontend:7233" // adjust to your frontend address
	pollInterval := 15 * time.Second

	conn, err := grpc.NewClient(frontendAddr, grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatalf("failed to connect: %v", err)
	}
	defer conn.Close()

	client := adminservice.NewAdminServiceClient(conn)

	ctx, cancel := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer cancel()

	ticker := time.NewTicker(pollInterval)
	defer ticker.Stop()

	log.Printf("starting DeepHealthCheck poller against %s, interval %s", frontendAddr, pollInterval)

	for {
		select {
		case <-ctx.Done():
			log.Println("poller stopped")
			return
		case <-ticker.C:
			pollCtx, pollCancel := context.WithTimeout(ctx, 10*time.Second)
			resp, err := client.DeepHealthCheck(pollCtx, &adminservice.DeepHealthCheckRequest{})
			pollCancel()
			if err != nil {
				log.Printf("DeepHealthCheck error: %v", err)
				continue
			}
			log.Printf("cluster state: %s", resp.State)
			for _, svc := range resp.Services {
				for _, host := range svc.Hosts {
					log.Printf("  service=%s host=%s state=%s", svc.Service, host.Address, host.State)
				}
			}
		}
	}
}
```

**Notes:**
- If TLS is required, replace `insecure.NewCredentials()` with your cluster's TLS configuration
- The polling interval determines metric freshness for alerting — set it based on your alerting SLA requirements
- `resp.State` is the aggregated fleet-level state evaluated by `healthCheckerImpl.Check` logic documented in section 2; the per-host detail is in `resp.Services[].Hosts[]`
- Before relying on `host_health` alerts, confirm what is currently calling `DeepHealthCheck` in your environment and at what interval

---

## 10. Quick reference — what to check when `host_health` alerts

```
host_health == 2 (NOT_SERVING) on N pods
│
├── Check frontend logs first
│   ├── "failed to ping deep health check" → connectivity issue, not threshold
│   ├── "health check exceeded host failure percentage threshold" → ≥50% pods NOT_SERVING
│   └── "health check exceeded host declined serving proportion threshold" → ≥5% pods DECLINED_SERVING
│
├── Check persistence latency/error metrics
│   └── Spiking? → persistence issue
│       ├── Check history pod logs for persistence errors
│       └── Investigate database layer at the same timestamp
│
└── Check history RPC latency metrics
    └── Spiking without persistence issues? → RPC/shard contention
        ├── Check shard acquisition logs
        └── Check membership change events around same timestamp

host_health == 2 (NOT_SERVING) on N pods with no persistence/RPC spike
│
│   (host_health = 2 during startup/shutdown is caused by gRPC health not SERVING,
│    which maps to NOT_SERVING after the 60s suppressStartupErrors init window expires)
│
├── Is a rolling restart / deploy in progress?
│   └── Yes → expected on pods mid-restart, monitor until pods return to 1
│
├── Are pods newly started and past the 60s init window?
│   └── Yes → shard acquisition still in progress; check acquisition logs
│
└── Pods stuck at 2 for > few minutes, no deploy in progress?
    ├── Check Kubernetes pod restart count → increasing = crashloop
    ├── Check pod logs for panic / shard acquisition errors
    └── Check for membership ring issues causing shard acquisition to stall

Majority of pods degraded AND infrastructure issue confirmed
└── Initiate manual namespace failover to secondary cluster
```