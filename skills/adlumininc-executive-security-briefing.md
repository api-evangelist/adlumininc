---
name: Adlumin executive security briefing
description: Assemble a weekly executive security summary from Adlumin's endpoint rollup, network health score and firewall aggregations.
api: openapi/adlumininc-api-openapi-original.yml
generated: '2026-09-07'
method: generated
source: >-
  Grounded in openapi/adlumininc-api-openapi-original.yml and the provider's "Executive Security
  Briefing" use case at https://developer.n-able.com/adlumin/page/examples-of-mcp-server-reporting
operations:
- GET /complete_endpoint_data
- GET /network_data
- GET /firewall
- GET /detections
mcp_tools:
- get_complete_endpoint_data
- get_network_data
- get_firewall_geo_aggregation
- get_detections
---

# Adlumin executive security briefing

Read-only. Four calls, one week window. Base `https://api.adlumin.com/v1`,
`Authorization: Bearer <token>`.

## 1. Endpoint posture — one call, not a scan

`GET /complete_endpoint_data?since=<ISO8601>&until=<ISO8601>`

This is the rollup endpoint and it exists precisely so you do not page through
`/endpoint_data`. It returns `tenant_id`, `generated_at`, and five blocks:
`at_risk_objects`, `stale_sensors_data`, `compliance_insights_data`, `network_health_data`,
`service_enablement_data`. Use it for the headline numbers.

## 2. Network health score

`GET /network_data?since=&until=` returns `network_health_score` (0–100, higher is healthier),
`last_calculated`, and a `stats[]` array of `{network_health_field, network_health_value}` pairs.

The fields are risk indicators — stale accounts, locked-out accounts, GPO violations,
unacknowledged high/critical detections, delegated and privileged domain accounts, service account
failures, circular groups. Quote the score, then name the two or three fields contributing most.

## 3. Firewall threat geography

`GET /firewall?aggregate=geo&since=&until=` returns `top_source_countries` and
`top_destination_countries`, each an array of `{country_code, event_count}`.

For repeat offenders: `GET /firewall?aggregate=blocked_ips&since=&until=` returns `top_blocked_ips`
with `source_address`, `block_count` and `month` (`YYYY-MM`) — the top 15 by month.

Do **not** page raw `/firewall` events for a briefing. The aggregations exist for this.

## 4. Detection volume

`GET /detections?since=&until=&severity=Critical,High` for the count. Read `total_count` from the
first page and stop — you do not need the rows.

## Rules

- Every number is tenant-scoped by the token. Say which tenant the briefing is for.
- `generated_at` and `last_calculated` are the freshness stamps. Print them; a stale score presented
  as live is the failure mode here.
- Nothing in this skill writes. If asked to "clear" or "acknowledge" anything, that is a different
  skill and it needs a human.
- No rate limit is published. Space the four calls rather than firing them in parallel bursts.
