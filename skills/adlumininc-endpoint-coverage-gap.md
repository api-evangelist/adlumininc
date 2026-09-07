---
name: Adlumin endpoint coverage gap check
description: Find registered devices with no healthy Adlumin-visible security agent by joining device inventory against agent telemetry.
api: openapi/adlumininc-api-openapi-original.yml
generated: '2026-09-07'
method: generated
source: >-
  Grounded in openapi/adlumininc-api-openapi-original.yml and the provider's "Endpoint Coverage Gap
  Check" use case at https://developer.n-able.com/adlumin/page/examples-of-mcp-server-reporting
operations:
- GET /device_data
- GET /endpoint_data
- GET /complete_endpoint_data
mcp_tools:
- get_device_data
- get_endpoint_data
- get_complete_endpoint_data
---

# Adlumin endpoint coverage gap check

Read-only. Base `https://api.adlumin.com/v1`, `Authorization: Bearer <token>`.

## Why two endpoints

The contract draws the distinction explicitly and it is the whole point of this skill:

- `GET /device_data` — **base inventory**. Every host registered with the tenant, *regardless of
  whether an agent is installed*. Fields: `hostname`, `ip_address`, `mac_address`,
  `operating_system`, `domain`, `last_seen`.
- `GET /endpoint_data` — **agent state**. Last-known telemetry reported by an installed agent
  (SentinelOne, Carbon Black, and so on). Fields: `hostname`, `ip_address`, `agent_type`,
  `agent_version`, `policy_name`, `online`, `last_seen`, `os`.

A host present in the first and absent from the second has no agent. A host present in both with
`online=false` has an agent that has stopped reporting. Those are different problems and the report
should say which is which.

## Procedure

1. Page all of `/device_data` (`per_page` max 100, walk until `page * per_page >= total_count`).
   Filters available: `domain`, `operating_system`, `search`.
2. Page all of `/endpoint_data`. Filters add `agent_type`, `online`, `since`, `until`.
3. **Join on `hostname`.** There is no shared id between the two collections — the join is a
   client-side string match. Normalize case and strip any domain suffix before comparing, and say so
   in the output: a hostname mismatch produces a false "no agent" finding, which is the main way this
   check goes wrong.
4. Bucket the result:
   - in `/device_data`, not in `/endpoint_data` → **no agent**
   - in both, `online=false` → **agent offline** (report `last_seen` and `agent_version`)
   - in both, `online=true`, old `agent_version` → **agent stale**

## Cross-check before you publish

`GET /complete_endpoint_data` returns the tenant rollup including `stale_sensors_data` and
`service_enablement_data`. If your counted gaps and the rollup disagree materially, trust the rollup
and say the join was inconclusive rather than reporting a number you derived by string matching.

## Rules

- Read-only; nothing here changes device or agent state.
- No rate limit is published, and this skill is the heaviest pager in the set — two full collections.
  Fetch sequentially, not in parallel.
- `401 unauthorized` mid-walk means the token expired during paging. Re-authenticate and restart the
  affected collection; partial pages must not be merged with pages from a different token session.
