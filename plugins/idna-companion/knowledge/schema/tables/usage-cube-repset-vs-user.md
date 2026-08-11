---
id: schema/tables/usage-cube-repset-vs-user
title: cqx_data.usage_cube_repset_vs_user
type: schema
status: verified
tags: [domain/idna, topic/usage]
related:
  - concepts/user-access-days
  - concepts/regulatory-usage-constraints
  - schema/tables/replica-set
  - schema/tables/department
  - schema/tables/location
  - schema/tables/usage-cube-repset-vs-department
  - schema/tables/usage-cube-repset-vs-location
  - schema/tables/repset-activity-summary-per-user
  - howtos/identify-content-creators-and-consumers
  - howtos/analyze-raw-user-activity
requires_capability: mcp_unrestricted
source: docs/reference/cqx_data_schema_enriched_all_tables.md
last_reviewed: 2026-05-08
---

# cqx_data.usage_cube_repset_vs_user

## Purpose

Application × user usage cross-reference. One row per (replica set, user). The per-user counterpart to [[schema/tables/usage-cube-repset-vs-department]] and [[schema/tables/usage-cube-repset-vs-location]]. Carries the read-vs-write split that distinguishes content **consumers** from **creators** for one application's heavy-user cohort. **Deny-listed in restricted mode** per [[concepts/regulatory-usage-constraints]].

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `replicaid` | text | Join key to `replica_set`. |
| `database_title` | text | Application title (denormalized). |
| `username` | text | Notes user name (real or pseudonymized depending on the customer's `pseudonymize` flag). |
| `department_id` | bigint | Join key to `department`. |
| `department_name` | text | Denormalized department name. |
| `location_id` | bigint | Join key to `location`. |
| `location_name` | text | Denormalized location name. |
| `last_access` | date | Most recent read by this user. |
| `last_access_category` | text | Bucketed recency label. |
| `days_since_last_access` | integer | Days since `last_access`. |
| `last_write_access` | date | Most recent write by this user. |
| `last_write_access_category` | text | Bucketed recency of last write. |
| `days_since_last_write_access` | integer | Days since `last_write_access`. |
| `days_active` | integer | Distinct days with access for this user-application pair. |
| `user_access_days` | integer | UAD for this (application, user) pair. |
| `sessions_total` | bigint | Total sessions. |
| `sessions_client` | bigint | Sessions from Notes client. |
| `sessions_web` | bigint | Sessions via browser/web channel. |
| `reads_client` | bigint | Client-side reads. |
| `writes_client` | bigint | Client-side writes. |

## Relationships

- `replicaid` -> `cqx_data.replica_set.replicaid`.
- `department_id` -> `cqx_data.department.department_id`.
- `location_id` -> `cqx_data.location.location_id`.
- The useful query grain is `(replicaid, username)`.

## Typical use

Heavy-user cohort for one application, ranked by UAD with the read/write split visible:

```sql
SELECT u.username,
  u.user_access_days,
  u.sessions_total,
  u.reads_client,
  u.writes_client,
  u.last_access,
  u.last_write_access,
  u.department_name,
  u.location_name
FROM cqx_data.usage_cube_repset_vs_user u
WHERE u.replicaid = '<replicaid>'
ORDER BY u.user_access_days DESC NULLS LAST
LIMIT 25
```

For the recipe wrapping this query (creators-vs-consumers classification, interpretation guidance), see [[howtos/identify-content-creators-and-consumers]].

## Notes and caveats

- **Deny-listed in restricted mode.** `cqx_role_mcp_restricted` has no `SELECT` on this table; semantic tools cannot reference it; `execute_sql` is unavailable under restricted mode (it is privileged-only). Reachable only when the active access profile is `privileged`. See [[concepts/regulatory-usage-constraints]] for the framing.
- For aggregate-grain alternatives that *are* allowlisted under rev 6.3, see [[schema/tables/usage-cube-repset-vs-department]] and [[schema/tables/usage-cube-repset-vs-location]].
- For a time-period split per user, see [[schema/tables/repset-activity-summary-per-user]].
- `username` carries the customer's chosen pseudonymization posture: hashed when `pseudonymize=Y`, real values when `pseudonymize=N`. The schema does not encode the policy — see `concepts/regulatory-usage-constraints`.

## Sources

CQX schema reference.
