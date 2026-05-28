# Temporal Server Dashboards

Grafana dashboards for monitoring a self-hosted Temporal Server cluster.

---

## Dashboards

### Temporal Server Dashboard — v2.4.0

Comprehensive cluster monitoring: throughput, persistence, service latencies, throttling, shard movement, workflow stats, matching, replication, and authorization.

| File | Description |
|---|---|
| [temporal-server.json](./temporal-server.json) | Grafana dashboard JSON — import this into Grafana |
| [temporal-server-readme.md](./temporal-server-readme.md) | Full panel reference and setup guide |
| [temporal-server-changelog.md](./temporal-server-changelog.md) | Version history |

---

### Temporal Standby Cluster — Replication Health — v2.1.0

Dedicated standby-side monitoring: replication lag, DLQ depth, standby task retries, stream health, and failover readiness.

| File | Description |
|---|---|
| [temporal-standby.json](./temporal-standby.json) | Grafana dashboard JSON — import this into Grafana |
| [temporal-standby-readme.md](./temporal-standby-readme.md) | Full panel reference and setup guide |
| [temporal-standby-changelog.md](./temporal-standby-changelog.md) | Version history |

---

### Temporal History Host Health Dashboard — v1.3.0

Deep health monitoring for the history service fleet: `host_health` state, gRPC readiness, persistence health, RPC health, and shard acquisition and movement.

| File | Description |
|---|---|
| [history-health-dashboard.json](./history-health-dashboard.json) | Grafana dashboard JSON — import this into Grafana |
| [history-health-dashboard-readme.md](./history-health-dashboard-readme.md) | Full panel reference and setup guide |
| [history-health-dashboard-changelog.md](./history-health-dashboard-changelog.md) | Version history |

---

## Multi-cluster filtering

All three dashboards include a `cluster` variable for filtering server metrics by the `clusterId` label (this is configurable, see below). Default selection is **All**, which matches every series — including series from setups that don't emit a cluster label at all.

**If you have a single cluster**, leave the defaults alone — the dashboards render identically to a non-filtered setup.

**If you have multiple clusters**, pick a value from the `cluster` dropdown to scope all panels to one cluster, or keep **All** to aggregate across them. If your metrics use a different label name (e.g. `cluster_id`, `cluster`), open **Dashboard settings → Variables → cluster_label** and set it to your label name.

### Caveat: `k8s_namespace` on the History Host Health dashboard with multiple clusters

The `k8s_namespace` dropdown is sourced from `kube_pod_status_ready` (kube-state-metrics), not Temporal, thus it may not include any cluster label. The dropdown therefore shows the union of Kubernetes namespaces across every cluster scraped by your Prometheus.

If you run multiple clusters and your kube-state-metrics **does** carry a cluster label, you can scope it manually by editing the variable query to add `{${cluster_label}=~"$cluster"}` just like the other variables.

---

## Updating to a new version

When a new version is available, reimport the updated JSON into Grafana via **Dashboards → Import**. Because the dashboard UID is stable across versions, Grafana will update your existing dashboard in place rather than creating a duplicate. Check the relevant changelog before updating to see what changed.