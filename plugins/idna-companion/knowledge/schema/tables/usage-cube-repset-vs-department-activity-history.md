---
id: schema/tables/usage-cube-repset-vs-department-activity-history
title: cqx_data.usage_cube_repset_vs_department_activity_history
type: schema
status: verified
tags: [domain/idna, topic/usage, topic/organization]
related:
  - schema/tables/usage-cube-repset-vs-location-activity-history
  - schema/tables/usage-cube-repset-vs-department
  - schema/tables/usage-cube-repset-activity-history
  - schema/tables/usage-cube-department-activity-history
  - schema/tables/department
  - schema/topics/department-and-location
  - concepts/user-access-days
  - concepts/usage-time-windows
requires_capability: none
source: ETL view definition (cqx_data_ifa, added 2026-07-28) + live validation
last_reviewed: 2026-08-24
---

# cqx_data.usage_cube_repset_vs_department_activity_history

## Purpose

Weekly activity history at **application × department** grain — the windowed
counterpart of [[schema/tables/usage-cube-repset-vs-department]] (which is
all-time totals). One row per (replicaid, department_id, ISO week). This is the
backing table for custom-window questions like "which departments have been most
active this year in app X?" and "top apps used by Sales this quarter" — the
`usage_over_window` cross mode reads it in both directions.

Available in **both** access profiles (restricted allowlist, org-attribution
aggregate category).

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `replicaid` | text | Application (replica set). |
| `database_title` | text | Application title (denormalized, carries a `[replicaid]` suffix). |
| `department_id` | bigint | Department; **`-1` = unattributed cohort** (see caveats). |
| `department_name` | text | Denormalized department name (threshold-pseudonymized at ETL). |
| `week_name` | text | ISO week label (`ISOYEAR-Wnn`). |
| `week_date` | date | Week anchor date (Monday). |
| `users_active` | integer | Distinct users of this app from this department that week. **Not additive across weeks.** |
| `days_active` | integer | Distinct active days that week. Additive. |
| `user_access_days` | integer | UAD that week. Additive across weeks. |
| `sessions_total` / `sessions_client` / `sessions_web` | bigint | Sessions that week. Additive. |
| `reads_client` / `writes_client` | bigint | Client-side reads/writes. Additive. |

## Relationships

- `replicaid` -> `cqx_data.replica_set.replicaid`.
- `department_id` -> `cqx_data.department.department_id` (except `-1`, which has
  no lookup row — it is a cohort, not an org unit).
- Location twin: [[schema/tables/usage-cube-repset-vs-location-activity-history]].

## Typical use

Departments active in one application over a custom window, ranked:

```sql
SELECT department_id, max(department_name) AS department,
  sum(user_access_days) AS uad, max(users_active) AS users_active_peak_week,
  sum(sessions_total) AS sessions
FROM cqx_data.usage_cube_repset_vs_department_activity_history
WHERE replicaid = '<replicaid>'
  AND week_date >= ('<from_date>'::date - 6) AND week_date <= '<to_date>'::date
GROUP BY department_id
ORDER BY uad DESC
```

Flip the grouping to `replicaid` with a `department_id` filter for "top apps
used by department Y".

## Notes and caveats

- **Department grain is direct membership (leaf)** — a user belongs to exactly
  one department; parent departments do NOT include their descendants here
  (unlike `usage_sessions_by_department`, which is a hierarchy roll-up).
  Subtree questions aggregate across the subtree's department_ids via
  `department.path @> ARRAY[<id>]`.
- **`department_id = -1` is real activity** that could not be attributed to any
  department (anonymous access, directory gaps). Keep it visible in rankings —
  it is a cohort, not noise, and it does not resolve in `cqx_data.department`.
- **`users_active` is never summed across weeks** (peak week only). Within one
  week it IS additive across sibling departments — disjoint user populations.
- Whole-week widening (custom windows cover every overlapping Monday-anchored
  week; see `covered_from_date` / `covered_to_date`) and instance-specific
  history depth apply as on
  [[schema/tables/usage-cube-repset-activity-history]].
- Below-threshold departments appear with pseudonymized names per iDNA's ETL
  threshold rule; rows and counts stay present.

## Sources

ETL view (added 2026-07-28) projecting the (replicaid × department × week)
grouping set of `dda_dm.dbusage_cube_activity_repset`; validated live the same
day. Motivated by the `usage_over_window` cross-mode performance finding — this
slice answers in ~60ms what the daily roll-up view needed 92s for.
