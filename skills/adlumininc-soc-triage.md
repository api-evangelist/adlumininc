---
name: Adlumin daily SOC triage
description: Pull unacknowledged Adlumin detections for a time window, rank them, and bulk-acknowledge only what a human has cleared.
api: openapi/adlumininc-api-openapi-original.yml
generated: '2026-09-07'
method: generated
source: >-
  Grounded in openapi/adlumininc-api-openapi-original.yml (GET /detections, POST
  /acknowledge_detections) and the provider's own "Daily SOC Triage" and "Batch Detection
  Acknowledgement" use cases at
  https://developer.n-able.com/adlumin/page/examples-of-mcp-server-reporting
operations:
- GET /detections
- POST /acknowledge_detections
mcp_tools:
- get_detections
- acknowledge_detections
---

# Adlumin daily SOC triage

Base URL `https://api.adlumin.com/v1`. Every request carries `Authorization: Bearer <token>`.
The token is tenant-scoped, so the tenant is never a parameter — you can only ever see one tenant.

## 1. Pull the active queue

`GET /detections` with:

- `acknowledged=false` — the active queue only
- `severity=Critical,High` — comma-separated; valid values are `Critical`, `High`, `Medium`, `Low`, `Informational`
- `since` / `until` — ISO 8601. For a 24-hour sweep, set `since` to now minus 24h.
- `page` / `per_page` — `per_page` caps at 100, default 25

The response is `{ total_count, page, per_page, data[] }`. There is **no cursor and no next-page
token**: walk pages by incrementing `page` until `page * per_page >= total_count`. Default sort is
newest-first by `event_time`.

## 2. Read the fields that matter

Each `Detection` carries `id` (e.g. `det_8a2f1c`), `severity`, `detection_type`, `event_time`,
`source_host`, `destination_host`, `account_used`, `information`, and an MDR workflow `status` from
the enum `Incident Declared`, `Request Customer Review`, `In Progress`, `Received By MDR`,
`Escalated`, `Rejected`.

Two flags decide whether a human is already on it:

- `cleared_from_abakis` — MDR has reviewed and cleared it.
- `corresponding_ticket` — a Jira URL, if one was raised.

**Do not acknowledge anything whose `status` is `Incident Declared`, `Escalated`, or
`Request Customer Review`.** Those are live MDR workflow states.

## 3. Acknowledge — only after a human decides

`POST /acknowledge_detections` with `{"detection_ids": ["det_8a2f1c", ...]}`. `detection_ids` is
required and must be a non-empty array; an empty one returns `400 bad_request` with
`"detection_ids must be a non-empty array"`.

Optional: `acknowledged_by` (defaults to the authenticated user) and `suppress_dashboard`
(default `false`).

The response splits the ids three ways: `acknowledged`, `not_found`, `already_acknowledged`. Report
all three back — `not_found` usually means a stale id from an earlier page.

## Rules

- **This is the only write on the whole API, and it has no undo.** No un-acknowledge operation exists
  and no `acknowledged=false` write is published. The detection survives — it stays queryable at
  `GET /detections?acknowledged=true` — but the dashboard state does not come back. Ask before
  bulk-acknowledging, and never set `suppress_dashboard=true` without an explicit instruction: it
  hides the detections tenant-wide from the MDR dashboard.
- **Retrying is safe.** Re-sending the same `detection_ids` returns them under
  `already_acknowledged` rather than applying twice. There is no `Idempotency-Key` header — this
  safety is a property of the operation, not of a key you control.
- **Errors** are `{ "error": "...", "message": "..." }`, not RFC 9457 problem+json. Expect
  `401 unauthorized` (missing or expired token — the API cannot tell you which),
  `400 bad_request`, `422 unprocessable_entity` (most often `since` after `until`).
- **No rate limit is published.** No `429`, no `Retry-After`, no `RateLimit-*` headers are documented.
  Throttle yourself: pace paging, and back off on any non-2xx you did not expect.
