# Changelog — Temporal Java SDK Dashboard (OTel)

## v1.1.0 — 2026-05-23

### Added
- **Workflow Task Heartbeat** panel in the _Workflow Task Info_ group — tracks `temporal_workflow_task_heartbeat`, the counter incremented each time the SDK forces a new workflow task because a local activity has consumed 80% of the workflow task timeout (`RespondWorkflowTaskCompleted` with `force_create_new_workflow_task=true`).
- Counter metrics (selected) table added to the _OTel Metric Naming Notes_ section of the readme, calling out `temporal_workflow_task_heartbeat` and related WFT counters.

---

## v1.0.0 — 2026-05-12

Initial versioned release.
