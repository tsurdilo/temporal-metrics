# Changelog — Temporal Standby Cluster — Replication Health Dashboard

## v2.1.0 — 2026-05-28

### Added
- **Multi-cluster filtering.** Two new template variables:
  - `cluster_label` (constant, default `clusterId`, hidden) — Prometheus label name identifying a cluster. Edit in dashboard settings if your metrics use a different label (e.g. `cluster_id`, `cluster`).
  - `cluster` (query, `label_values(${cluster_label})`, includeAll, allValue `.*`, default **All**) — filters every panel by cluster.
- All panel selectors now include `${cluster_label}=~"$cluster"`. Backward compatible: with **All** selected the expansion is `=~".*"`, which matches series even when the cluster label is absent — single-cluster setups with no `clusterId` label render identically to v2.0.0 with no configuration needed.

### Changed
- `namespace` and `source_cluster` variable queries scoped to the selected cluster (`{${cluster_label}=~"$cluster"}`). Prevents values from other clusters appearing in the dropdowns.

---

## v2.0.0 — 2026-05-12

First versioned release. Prior changes were unversioned.
