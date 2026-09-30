# Starter n8n Workflow Map

This file describes the starter `n8n` workflow layout for:

- scheduled `dealstage` batch intake
- AI enhancement orchestration
- review-triggered reruns from `reviewgate`

`n8n` runs on VPS2 (moved from beside `reviewgate` on VPS1, 2026-09-30).
It reaches VPS1 services (reviewgate, MySQL) over the WireGuard tunnel at
`10.20.0.1`, and aistage locally. None of these workflows exist yet: this is
the starting plan.

Important framing:

- `aistage` is a shared AI utility
- `n8n` is only one possible caller of that utility
- `dealstage` and `reviewgate` may also call it directly when appropriate

Store exported workflow JSON files under:

- [workflows](workflows)

## Workflow Set

Use these starter workflows:

1. `scheduled_amzn_intake`
2. `scheduled_amzn_ai_enhancement`
3. `reviewgate_rerun_stage`
4. `reviewgate_rerun_full_pipeline`
5. `reviewgate_alt_endpoint_compare`
6. `reviewgate_failure_notify`

## 1. `scheduled_amzn_intake`

Purpose:

- start scheduled work in `dealstage`
- freeze the next candidate batch
- kick off the AI enhancement workflow

Suggested trigger:

- Cron node

Suggested schedule:

- every 30 minutes for intake
- or nightly if you want a single larger run

Suggested node chain:

1. `Cron`
2. `HTTP Request` -> `POST dealstage /api/batches/start`
3. `HTTP Request` -> `POST dealstage /api/batches/{batch_id}/fetch-candidates`
4. `IF` source rows > 0
5. `Execute Workflow` -> `scheduled_amzn_ai_enhancement`
6. `Else` -> no-op log branch

## 2. `scheduled_amzn_ai_enhancement`

Purpose:

- run the initial AI enhancement process
- keep `dealstage` as source of truth
- call `aistage` in sequence as the shared AI utility

Suggested node chain:

1. `Execute Workflow Trigger`
2. `HTTP Request` -> `POST dealstage /api/ai-runs/start`
3. `HTTP Request` -> `GET dealstage /api/review-queue/next?status=PENDING_AI&limit=5`
4. `Split In Batches`
5. `HTTP Request` -> `POST aistage /v1/stages/outline_angle`
6. `HTTP Request` -> `POST dealstage /api/review-queue/{id}/stage-result`
7. `HTTP Request` -> `POST aistage /v1/stages/draft_intro`
8. `HTTP Request` -> `POST dealstage /api/review-queue/{id}/stage-result`
9. `HTTP Request` -> `POST aistage /v1/stages/draft_deep_dive`
10. `HTTP Request` -> `POST dealstage /api/review-queue/{id}/stage-result`
11. `HTTP Request` -> `POST aistage /v1/stages/draft_comparison`
12. `HTTP Request` -> `POST dealstage /api/review-queue/{id}/stage-result`
13. `HTTP Request` -> `POST aistage /v1/stages/draft_faq`
14. `HTTP Request` -> `POST dealstage /api/review-queue/{id}/stage-result`
15. `HTTP Request` -> `POST aistage /v1/stages/humanizer_compliance`
16. `HTTP Request` -> `POST dealstage /api/review-queue/{id}/stage-result`
17. `HTTP Request` -> `POST dealstage /api/review-queue/{id}/mark-ready`
18. optional `HTTP Request` or `Discord` notification

## Initial AI Enhancement Process

This is the baseline sequence the workflow should run:

1. `outline_angle`
2. `draft_intro`
3. `draft_deep_dive`
4. `draft_comparison`
5. `draft_faq`
6. `humanizer_compliance`
7. mark item `PENDING_REVIEW`

That matches the current design in:

- `dealstage`
- `aistage`
- `reviewgate`

## 3. `reviewgate_rerun_stage`

Purpose:

- handle a reviewer’s request to rerun a specific stage

Suggested trigger:

- Webhook node

Webhook payload fields:

- `review_queue_id`
- `candidate_id`
- `actor_email`
- `rerun_target`
- `model_override`
- `endpoint_override`

Suggested node chain:

1. `Webhook`
2. `HTTP Request` -> `POST dealstage /api/review-queue/{id}/request-rerun`
3. `HTTP Request` -> `GET dealstage /api/review-queue/{id}`
4. `Switch` by `rerun_target`
5. `HTTP Request` -> target `aistage` stage endpoint
6. `HTTP Request` -> `POST dealstage /api/review-queue/{id}/stage-result`
7. `HTTP Request` -> `POST dealstage /api/review-queue/{id}/mark-ready`
8. optional `HTTP Request` back to `reviewgate` internal callback or just rely on refresh

## 4. `reviewgate_rerun_full_pipeline`

Purpose:

- rerun all AI stages for a single review item

Suggested trigger:

- Webhook node

Suggested node chain:

1. `Webhook`
2. `HTTP Request` -> `POST dealstage /api/review-queue/{id}/request-rerun`
3. `HTTP Request` -> `GET dealstage /api/review-queue/{id}`
4. run:
   - `outline_angle`
   - `draft_intro`
   - `draft_deep_dive`
   - `draft_comparison`
   - `draft_faq`
   - `humanizer_compliance`
5. persist each result to `dealstage`
6. mark ready again

## 5. `reviewgate_alt_endpoint_compare`

Purpose:

- send a review item to an alternate model or alternate worker endpoint
- preserve comparison against the prior saved result

Suggested trigger:

- Webhook node

Suggested node chain:

1. `Webhook`
2. `HTTP Request` -> `GET dealstage /api/review-queue/{id}`
3. `HTTP Request` -> alternate `aistage` or provider endpoint
4. `HTTP Request` -> `POST dealstage /api/review-queue/{id}/stage-result`
5. optional `HTTP Request` -> store comparison metadata

## 6. `reviewgate_failure_notify`

Purpose:

- send notifications on:
  - failed AI stage
  - failed scheduled batch
  - failed rerun

Suggested trigger:

- Error Trigger
- or an `IF` branch in the main workflows

Suggested actions:

- Discord webhook
- email
- internal ops log

## Suggested Credentials In n8n

Create credentials for:

- `dealstage_private_api`
- `aistage_private_api`
- `discord_exception_webhook` optional

## Suggested Variables

Use environment variables or workflow variables for:

- `DEALSTAGE_API_BASE_URL`
- `DEALSTAGE_API_TOKEN`
- `AISTAGE_API_BASE_URL`
- `AISTAGE_API_TOKEN`
- `REVIEWGATE_CALLBACK_BASE_URL` optional

## Practical v1 Recommendation

Stand up these first:

1. `scheduled_amzn_intake`
2. `scheduled_amzn_ai_enhancement`
3. `reviewgate_rerun_stage`

Then add:

4. `reviewgate_rerun_full_pipeline`
5. `reviewgate_alt_endpoint_compare`
6. `reviewgate_failure_notify`
