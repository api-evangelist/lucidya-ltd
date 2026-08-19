---
name: lucidya-pull-social-listening-widget-data
description: >-
  Pull analytics widget data for a Lucidya Social Listening monitor — discover the
  account's monitors, find the widgets available on a page, then request widget
  data for a date range from X (Twitter), Facebook, Instagram or news/blogs.
api: Lucidya Social Listening API
base_url: https://api.lucidya.com
spec: ../openapi/lucidya-ltd-social-listening-api-openapi.yml
generated: '2026-08-13'
method: generated
source: openapi/lucidya-ltd-social-listening-api-openapi.yml + docs.lucidya.com introduction articles
operations:
  - monitors_list
  - pages
  - Widgets
  - filters
  - post-twitter-widget-data
  - post-facebook-widget-data
  - post-instagram-widget-data
  - post-new-blogs-widget-data
---

# Pull Social Listening widget data

Every call carries the header `luc-authorization: <token>`. The token must be a
**Social Listening** API key — Lucidya binds each key to one API product type, so
an AI or OmniServe key returns 401 here. See
`../authentication/lucidya-ltd-authentication.yml`.

## 1. List the monitors

`monitors_list` — `GET /monitors_list`

- Required query: `page_id` (integer, starts at 1). It is **required**; omitting it
  returns `400 bad_request` with detail `Page_id is required`.
- The response carries a pagination block `{"page_number": "1", "count": <total pages>}`.
  Pages are fixed at 10 results. Increment `page_id` until `page_number` reaches `count`.

Keep the `id` of the monitor you want. It is the `monitor_id` every later call needs.

## 2. Find the pages and widgets on that monitor

`pages` — `GET /pages`
`Widgets` — `GET /widgets`

`Widgets` requires all four of `monitor_id`, `product_id`, `page_name` and
`data_source`. Do not guess `page_name` or `data_source`; read them from the
`pages` response. `widgets_names` in step 4 must be names this call returned.

## 3. Optionally read the filter set

`filters` — `GET /filters`

Lucidya does not accept an ad-hoc filter grammar. You retrieve the permitted
filter set from this endpoint and post the values back with the widget request.

## 4. Request the widget data

Pick the operation matching the data source:

| Data source | Operation | Path |
|---|---|---|
| X (Twitter) | `post-twitter-widget-data` | `POST /twitter/widget_data` |
| Facebook | `post-facebook-widget-data` | `POST /facebook/widget_data` |
| Instagram | `post-instagram-widget-data` | `POST /instagram/widget_data` |
| News & blogs | `post-new-blogs-widget-data` | `POST /nb/widget_data` |

All four take the same required query parameters:

- `monitor_id` (integer) — from step 1
- `page_name` (string) and `data_source` (string) — from step 2
- `start_date`, `end_date` (integer) — **Unix timestamps in seconds**, not ISO dates
- `widgets_names` (string) — widget names from step 2

## Handling the response

- `200` — success. The envelope is `{"status":200,"success":true,"data":{...}}`.
- `400 bad_request` — invalid parameters, fields or filters. Most often a missing
  `page_id` or a `widgets_names` value that does not exist on that page.
- `401 unauthorized` — invalid key. Also what you get when the key was idle 30 days
  (deactivated) or has been in continuous use for 60 days (expired), or when the
  calling IP is outside the subnet allow-list registered on the key.
- `403 forbidden` — access not allowed, **or** the caller has been blocked after too
  many errors in a short window. Stop and investigate; do not retry into a 403.
- `429 too_many_requests` — rate limit exceeded. The floor is 100 requests/minute and
  the window resets at the top of the next minute. **No `Retry-After` or `RateLimit-*`
  header is returned**, so sleep until the next minute boundary and resume.
- `500` / `503` / `504` — declared on nearly every operation. `504` is the most widely
  declared failure in the whole surface because these are aggregation queries. Retry
  with backoff; narrow the date range if it persists.

Errors use `{"error":{"status":<int>,"detail":"<message>"}}` — not RFC 9457
problem+json. Full registry: `../errors/lucidya-ltd-problem-types.yml`.

## Rules

- There is **no idempotency key** on any Lucidya operation. A retried POST is a
  second request, not a replay.
- Do not send `page_number` as a request parameter. It is a response field only;
  the request parameter is `page_id`.
- Lucidya's Authorization article spells the header `luc-autharization` in one prose
  sentence. That is a typo in their docs. The header is `luc-authorization`.
