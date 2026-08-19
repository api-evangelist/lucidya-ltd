---
name: lucidya-run-omniserve-analytics-job
description: >-
  Run a Lucidya OmniServe Analytics report — discover analytics pages, read the
  widgets and filters a page supports, create an analytics job over a date range,
  then poll for the rendered result. Also covers CSAT/in-chat-survey analytics and
  the agent, team, SLA, routing and data-source reference lookups.
api: Lucidya OmniServe Analytics API
base_url: https://api.lucidya.com/public_api/omniserve
spec: ../openapi/lucidya-ltd-omniserve-analytics-api-openapi.yml
generated: '2026-08-13'
method: generated
source: openapi/lucidya-ltd-omniserve-analytics-api-openapi.yml
operations:
  - getAnalyticsPages
  - getPageWidgets
  - getPageFilters
  - createAnalyticsJob
  - getCsatQuestions
  - csatResponseCounts
  - csatQuestionResponses
  - csatUserResponses
  - createCustomFieldsJob
  - getCustomFieldsResults
  - getAgents
  - getTeams
  - getDataSources
  - getRoutings
  - getSlas
---

# Run an OmniServe Analytics job

OmniServe is the one Lucidya API on a sub-path base URL:
`https://api.lucidya.com/public_api/omniserve`. Prefixing paths onto
`https://api.lucidya.com` alone will 404.

It is also the only Lucidya spec that **declares** its security scheme —
`OmniserveToken`, an apiKey in the `luc-authorization` header. Every operation
requires it.

Access is plan-gated: the published pricing page grants "OmniServe limited APIs" on
Growth and "OmniServe full APIs" on Enterprise, without saying which operations fall
on which side. Expect `403 forbidden` on operations outside your tier.

## 1. Discover pages, widgets, filters

`getAnalyticsPages` — `GET /analytics/pages`
`getPageWidgets` — `GET /analytics/{page_name}/widgets`
`getPageFilters` — `GET /analytics/{page_name}/filters`

`page_name` is a path parameter taken from the pages response. Widget names from
`getPageWidgets` are the only valid values for `widgets_names` in step 2.

## 2. Create the job

`createAnalyticsJob` — `POST /analytics/{page_name}/create`

Two body encodings are declared. Pick one:

**`application/json`**
```json
{
  "widgets_names": ["Inbox Overview", "Engagements Volume"],
  "start_date": 1640995200,
  "end_date": 1643673600
}
```

**`application/x-www-form-urlencoded`** — adds `monitors` (comma-separated monitor
ids, e.g. `45930,45922`) and `filters` (a **URL-encoded JSON string**, not a nested
object).

Dates are Unix timestamps in seconds.

## 3. Poll for the result

`GET /analytics/{page_name}/index`

This operation has **no `operationId`** in the spec — address it by path.

- `202` — job still running. Poll again.
- `200` — result ready.

Same create-then-poll shape as the OmniChannel widget endpoints and the AI audio flow.

## CSAT / in-chat survey

- `getCsatQuestions` — `GET /analytics/in-chat-survey/csat_questions`
- `GET /analytics/in-chat-survey/filters` (no operationId)
- `GET /analytics/in-chat-survey/index` (no operationId) — the poll endpoint, returns 202 while pending
- `csatResponseCounts` — `POST /analytics/in-chat-survey/csat_response_counts`
- `csatQuestionResponses` — `POST /analytics/in-chat-survey/csat_question_responses`
- `csatUserResponses` — `POST /analytics/in-chat-survey/csat_user_responses`

## Custom fields

`createCustomFieldsJob` — `POST /analytics/custom_fields`, then
`getCustomFieldsResults` — `GET /analytics/custom_fields` (202 while pending).

## Reference lookups

`getAgents`, `getTeams`, `getDataSources`, `getRoutings`, `getSlas` — the dimensions
you slice analytics by. Fetch these first and cache them; they change rarely and each
call spends rate-limit budget.

## Errors

OmniServe declares the fullest error surface in the platform — `400`, `401`, `403`,
`404`, `405`, `406`, `410`, `422`, `429`, `500`, `503`, `504` on every operation.

The OmniServe error envelope differs from the other products: `Error` is
`{ "error", "status", "message", "code" }`, and the live mock returns
`{"status":401,"message":"Authentication required"}` — a `message` field, not
`error.detail`. Handle both.

## Rules

- `202` is not success. Branch on it explicitly.
- No idempotency key: a retried `createAnalyticsJob` starts a second job.
- 100 requests/minute floor, fixed window, no `Retry-After`. Poll on a timer, not a
  tight loop.
