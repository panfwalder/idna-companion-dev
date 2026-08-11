---
id: schema/tables/user-activity-summary
title: cqx_data.user_activity_summary
type: schema
status: verified
tags: [domain/idna, topic/usage]
related:
  - schema/tables/usage-sessions
  - schema/tables/usage-cube-repset-vs-user
  - schema/tables/department
  - schema/tables/location
  - concepts/regulatory-usage-constraints
requires_capability: mcp_unrestricted
source: ETL view definition (cqx_data_ifa) + dda_dm build SQL, verified 2026-07-27
last_reviewed: 2026-07-27
---

# cqx_data.user_activity_summary

## Purpose

Person-grain activity summary — one row per known user, linking the Domino directory
(person documents) with observed activity. This is the resolution table between
`person_id` (as carried by [[schema/tables/usage-sessions]]) and `username`, and the
source of a user's department/location attribution.

**Deny-listed in restricted mode** (per-user data). Reachable only under the
privileged profile.

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `person_id` | bigint | Node ID; **NULL when the underlying ID is ≤ 0** (unresolved/anonymous). Join key to `usage_sessions.person_id`. |
| `username` | text | Notes user name (pseudonymized when `pseudonymize=Y`). |
| `has_person_document` | boolean | User exists in the directory snapshot (`snp_user_documents`). |
| `has_shown_activity` | boolean | User has observed sessions. |
| `last_activity` | date | Most recent session. |
| `last_activity_days_ago` | integer | Days since. |
| `last_activity_focus_db` | date | Most recent session on a focus DB. |
| `category_activity_timeframe` | text | Bucketed recency label. |
| `count_dbs_used` / `count_focus_dbs_used` | integer | Distinct databases used. |
| `count_replica_sets_used` / `count_focus_replica_sets_used` | integer | Distinct replica sets used. |
| `category_db_usage` / `category_focus_db_usage` / `category_focus_rs_usage` | text | Bucketed breadth-of-use labels. |
| `department_id` | bigint | Join key to `department`. **Nullable — see caveats.** |
| `location_id` | bigint | Join key to `location`. **Nullable — see caveats.** |

## Relationships

- `person_id` -> `cqx_data.usage_sessions.person_id` (the only path from session
  rows to usernames).
- `department_id` -> `cqx_data.department.department_id`;
  `location_id` -> `cqx_data.location.location_id`.
- One row per username; the build keeps `min(client_node_id)` per username.

## Typical use

Resolve usernames and org attribution when aggregating raw sessions per user — see
the LEFT JOIN pattern discussion in the caveats. The full validated recipe lives in
the raw user-activity howto.

## Notes and caveats

- **Always LEFT JOIN this table from `usage_sessions`** — an inner join silently
  drops anonymous web sessions (no `person_id`; ~20% of web session volume on the
  reference instance) and any session under a node ID this table does not carry.
- **`department_id` / `location_id` are NULL for users without a person-document org
  mapping** — coverage equals the customer's directory hygiene. Inner joins to
  `department`/`location` silently drop those users. On instances with thin
  directories the drop can be the majority of activity.
- **Eval/trial license NULLs `department_id`/`location_id` for every user** (config
  default is eval=true when the flag is absent). Queries with inner org joins return
  empty results on demo instances. Note this is a different mechanism from the usage
  cube, which masks *names* under eval license while keeping IDs.
- **One department/location per user, by construction** — the build takes
  `min(org_unit_id)` per username; attribution is a current snapshot applied to all
  historical activity, not a per-session fact.
- A user's sessions recorded under non-primary name variations or non-min node IDs
  do not resolve to this row — they surface as unresolved when joining from
  `usage_sessions`.

## Sources

CQX view over `dda_dm.user_activity_summary`; build SQL
(`240_dda_dm.user_activity_summary.update.sql`) verified 2026-07-27 for the
person_id, org-sourcing, and eval-license semantics above.
