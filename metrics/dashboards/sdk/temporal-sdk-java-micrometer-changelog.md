# Changelog — Temporal Java SDK Dashboard (Micrometer)

## v1.1.0 — 2026-05-23

### Added
- **Workflow Task Heartbeat** panel in the _Workflow Task Info_ group — tracks `temporal_workflow_task_heartbeat_total`, the counter incremented each time the SDK forces a new workflow task because a local activity has consumed 80% of the workflow task timeout (`RespondWorkflowTaskCompleted` with `force_create_new_workflow_task=true`).
- `temporal_workflow_task_heartbeat` → `temporal_workflow_task_heartbeat_total` added to the counter metrics naming table in the readme.

---

## v1.0.0 — 2026-05-12

Initial versioned release.
