---
name: lucidya-manage-cdp-profiles-and-segments
description: >-
  Work with unified customer records in the Lucidya CDP — list and read profiles,
  create and update them, pull a profile's interactions and survey responses, and
  add or remove profiles from a segment.
api: Lucidya CDP API
base_url: https://api.lucidya.com
spec: ../openapi/lucidya-ltd-cdp-api-openapi.yml
generated: '2026-08-13'
method: generated
source: openapi/lucidya-ltd-cdp-api-openapi.yml + https://docs.lucidya.com/docs/cdp-api/rc9aova8s2jii-pagination
operations:
  - getProfileList
  - getProfileById
  - createProfile
  - updateProfile
  - getFilters
  - getSurveyByProfileId
  - getInteractionProfileById
  - createInteractionProfile
  - getSegmentList
  - appendProfilleToSegment
  - deleteProfilleFromSegment
---

# Manage CDP profiles and segments

The CDP is the only Lucidya product with a real write surface. It is also the one
holding personal data, under both GDPR and the Saudi PDPL — see
`../conformance/lucidya-ltd-conformance.yml`.

Header on every call: `luc-authorization` with a **CDP-type** key.

## Read profiles

`getProfileList` — `GET /profiles`

- `start_date` and `end_date` (integer Unix timestamps) are **required**. There is no
  unbounded list; you always list within a window.
- `page` (integer, optional) walks the result set. Batches are 10 records; the response
  carries `{"page_number": "...", "count": <total pages>}`.

`getProfileById` — `GET /profiles/{id}` for a single record.

`getFilters` — `GET /profiles/filters` returns the permitted filter set. Retrieve it
before filtering rather than composing filters by hand.

## Write profiles

`createProfile` — `POST /profiles`
`updateProfile` — `PUT /profiles`

Note the shape: update is a `PUT` on the **collection**, not on `/profiles/{id}`. The
identifier travels in the body.

**There is no DELETE.** The API can create and update a customer profile but cannot
remove one. Any erasure request under GDPR Article 17 or the PDPL must be handled
outside this API.

## Interactions and surveys

`getInteractionProfileById` — `GET /profiles/{id}/interactions`
`createInteractionProfile` — `POST /profiles/interactions`
`getSurveyByProfileId` — `GET /profiles/{id}/surveys`

Both hang off a profile. Interactions are created against the collection path, with the
profile reference in the body.

## Segments

`getSegmentList` — `GET /segments`
- Optional `page`, `sort_by`, `order_by`.

`appendProfilleToSegment` — `PUT /segments/append_profiles`
- Required: `segment_id` (integer), `profiles_ids` (array).

`deleteProfilleFromSegment` — `DELETE /segments/delete_profiles`
- Same parameters.

Segment membership is mutated through these two dedicated endpoints; you never PATCH
the segment resource itself. Both operation ids are misspelled ("Profille") in
Lucidya's spec — use them exactly as written, they are the real ids.

## Errors

`400 bad_request`, `401 unauthorized`, `403 forbidden`, `404 not_found`,
`406 not_acceptable`, `422 validation_failed`, `500`, `503`, `504`. Envelope:
`{"error":{"status":<int>,"detail":"<message>"}}`.

`getFilters` also declares a `306` response. 306 is a reserved, unused HTTP status —
treat it as a spec authoring error, not as a condition to branch on.

## Rules

- No idempotency key. A retried `POST /profiles` after a network timeout can create a
  duplicate customer record. Read back with `getProfileList` over a narrow window
  before retrying a create.
- Rate limit floor is 100 requests/minute, resetting on the minute. No `RateLimit-*` or
  `Retry-After` headers are returned.
- The CDP documentation project has not been updated since 2024-08-18, the oldest of
  the five. Verify behaviour against a live response before relying on a detail.
