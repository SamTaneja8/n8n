# n8n Workflow Artifacts

This folder is mounted into the `n8n` container at:

- `/workflows`

Suggested use:

- keep exported workflow JSON files here
- keep import-ready starter workflow snapshots here
- keep environment-specific notes alongside the exports

Suggested filenames once you begin exporting real workflows:

- `scheduled_amzn_intake.json`
- `scheduled_amzn_ai_enhancement.json`
- `reviewgate_rerun_stage.json`
- `reviewgate_rerun_full_pipeline.json`
- `reviewgate_failure_notify.json`

Use [N8N_WORKFLOW_MAP.md](../../N8N_WORKFLOW_MAP.md) as the planning guide for those exports.
