---
name: Adlumin Active Directory hygiene audit
description: Audit at-risk Active Directory groups, shares and systems plus compliance insight counters, separating live risk from granted exemptions.
api: openapi/adlumininc-api-openapi-original.yml
generated: '2026-09-07'
method: generated
source: >-
  Grounded in openapi/adlumininc-api-openapi-original.yml and the provider's "Active Directory
  Hygiene Audit" and "Compliance Posture Reporting" use cases at
  https://developer.n-able.com/adlumin/page/examples-of-mcp-server-reporting
operations:
- GET /at_risk_groups
- GET /at_risk_shares
- GET /at_risk_systems
- GET /compliance_insights
mcp_tools:
- get_at_risk_groups
- get_at_risk_shares
- get_at_risk_systems
- get_compliance_insights
---

# Adlumin Active Directory hygiene audit

Read-only. Base `https://api.adlumin.com/v1`, `Authorization: Bearer <token>`.

## The exemption trap — read this first

All three at-risk endpoints **exclude exempted objects by default**. Each takes a `type` parameter
that switches the result set between currently at-risk objects and objects in an exempted state.
An audit that only calls the default view reports a shrinking risk surface when what actually
happened is that somebody granted an exemption. **Always run both passes and report the exempted
count alongside the at-risk count.**

## 1. Groups

`GET /at_risk_groups` — Active Directory groups flagged for overly broad permissions, privileged
access or policy violations. Filters: `type`, `privileged`, `search`, plus `page` / `per_page` /
`sort_column` / `sort_dir`.

Each `AtRiskGroup` carries `group_name`, `domain`, `privileged`, `at_risk`, `exclusion`,
`is_domain_group`, and four membership counters: `account_members_count`, `computer_members_count`,
`group_members_count`, `group_member_of_count`. A high `group_member_of_count` is the nesting signal
that produces circular groups.

## 2. Shares

`GET /at_risk_shares` — network shares with overly permissive access controls (world-readable or
writable, or reachable by privileged groups). Fields: `share_name`, `share_path`, `host_id`,
`hostname`, `at_risk`, `exclusion`, `note`.

`host_id` is a host reference, but there is no get-a-system-by-id operation, so resolve a share to
its machine by matching `hostname` against `/at_risk_systems` or `/device_data`.

## 3. Systems

`GET /at_risk_systems` — hosts flagged for misconfiguration, stale patches, privileged account
exposure or anomalous activity. Filters add `operating_system` and `domain`. Fields: `hostname`,
`ip_address`, `operating_system`, `domain`, `mac_address`, `at_risk`, `exclusion`, `note`.

## 4. Compliance counters

`GET /compliance_insights?since=&until=` returns the audit-facing integers:

| Field | Meaning |
|---|---|
| `stale_accounts_count` | AD accounts inactive for 90+ days |
| `password_never_expire_count` | Accounts with password expiry disabled |
| `password_reversible_encryption_count` | Accounts storing passwords with reversible encryption |
| `stale_passwords_count` | Passwords not changed in 90+ days |
| `gpo_violations_count` | Active Group Policy Object violations |

Plus `network_health_stats[]` of `{network_health_field, network_health_value}` for the granular
metrics — IT Operations Failures, Privileged Domain Accounts, Circular Groups and similar.

## Rules

- Pagination is `page` / `per_page` (max 100) with `total_count`. No cursor. Walk pages until
  `page * per_page >= total_count`.
- These counters are point-in-time. Stamp the report with the window you passed in `since`/`until`
  and do not present a snapshot as a trend unless you fetched more than one window.
- Nothing here writes. There is no exemption-granting operation in the public API — exemptions are
  set in the platform, not through this contract.
- Errors are `{error, message}`; `422 unprocessable_entity` most often means `since` is after `until`.
