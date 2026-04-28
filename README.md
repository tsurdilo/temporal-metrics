# Temporal Server Operations

This repository includes various Temporal Server operational artifacts.

---

## [Dynamic Config](dynamic_config/README.md)

OSS Temporal server dynamic config reference, dynamic config YAML samples, and troubleshooting info.

---

## [Metrics](metrics/references/README.md)

OSS Temporal server metrics references.

### Dashboards

- [Server Dashboards](metrics/dashboards/server/README.md) — Grafana dashboards for monitoring a self-hosted Temporal Server cluster, including:
    - [Server Overview Dashboard](metrics/dashboards/server/temporal-server-readme.md) — cluster health, throughput, persistence, and service metrics
    - [Standby Cluster Dashboard](metrics/dashboards/server/temporal-standby-readme.md) — replication health, lag, and failover readiness for standby clusters
- [SDK Dashboards](metrics/dashboards/sdk/README.md) — Grafana dashboards for monitoring Temporal SDK clients and workers (Java, Go, TypeScript, Python, .NET, Ruby).
- [Troubleshooting Dashboards](metrics/dashboards/troubleshooting/README.md) — Grafana dashboards focused on troubleshooting specific Temporal operational issues.