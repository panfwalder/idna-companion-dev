---
id: schema/tables/usage-cube-repset-activity-history
title: cqx_data.usage_cube_repset_activity_history
type: schema
status: verified
tags: [domain/idna, topic/usage]
related:
  - schema/tables/replica-set
  - schema/tables/usage-cube-weekly-summary
  - schema/tables/usage-cube-department-activity-history
  - schema/tables/usage-cube-location-activity-history
  - concepts/user-access-days
  - concepts/usage-tier
requires_capability: none
source: ETL view definition (cqx_data_ifa) + live verification 2026-07-27
last_reviewed: 2026-07-27
---

# cqx_data.usage_cube_repset_activity_history

## Purpose

Weekly activity history **per application** (replica set). One row per
`(replicaid, week_date)`, fully aggregate — no user, department, or location
dimension. This is the workhorse for custom-time-window questions at application
grain ("how active was app X in the last six months?") and for trend analysis
("is usage of app X rising or falling?").

Available in **both** access profiles — this table is on the restricted allowlist,
unlike the per-user usage tables.

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `replicaid` | text | Join key to `replica_set`. |
| `database_title` | text | Application title (denormalized). |
| `week_name` | text | ISO week label (`ISOYEAR-Wnn`, e.g. `2026-W12`). |
| `week_date` | date | Week anchor date (Monday of the ISO week). |
| `departments_active` | integer | Distinct departments active that week. |
| `locations_active` | integer | Distinct locations active that week. |
| `users_active` | integer | Distinct active users that week. **Not additive across weeks.** |
| `days_active` | integer | Distinct active days that week. Additive across weeks. |
| `user_access_days` | integer | UAD that week. Additive across weeks. |
| `sessions_total` | bigint | Total sessions that week. Additive. |
| `sessions_client` | bigint | Notes-client sessions that week. Additive. |
| `sessions_web` | bigint | Web sessions that week. Additive. |
| `reads_client` | bigint | Client-side reads that week. Additive. |
| `writes_client` | bigint | Client-side writes that week. Additive. |

## Relationships

- `replicaid` -> `cqx_data.replica_set.replicaid`.
- Weekly twin of the department/location activity histories
  ([[schema/tables/usage-cube-department-activity-history]],
  [[schema/tables/usage-cube-location-activity-history]]) and the estate-level
  [[schema/tables/usage-cube-weekly-summary]].

## Typical use

Activity of one application over a custom window (e.g. the last six months):

```sql
SELECT
  sum(user_access_days) AS uad,
  sum(sessions_total)   AS sessions,
  sum(reads_client)     AS reads,
  sum(writes_client)    AS writes,
  max(users_active)     AS users_active_peak_week,
  count(*)              AS weeks_covered
FROM cqx_data.usage_cube_repset_activity_history
WHERE replicaid = '<replicaid>'
  AND week_date >= current_date - 183
```

Weekly trend for the same app: select `week_date, user_access_days, users_active`
ordered by `week_date`.

## Notes and caveats

- **`users_active` must never be summed across weeks** — it is a per-week distinct
  count; the same user recurs in many weeks. `MAX` gives the peak week, which is a
  floor, not the window-distinct headcount. Exact distinct users over an arbitrary
  window require per-user data (privileged profile only).
- **All other metrics are genuinely additive across weeks**: weeks partition dates,
  so distinct (date, user) pairs in different weeks are disjoint — summing
  `user_access_days` over a window is exact.
- **ISO-week floor.** Weeks are Monday-anchored ISO weeks; a custom window's edges
  land mid-week, so the first and last covered weeks are partial. State this when
  reporting a window total.
- **History depth is instance-specific** — there is no retention cut in the ETL;
  depth equals source-table retention. Check `min(week_date)` before promising a
  window.
- **No tier claims from custom windows.** Usage tiers ([[concepts/usage-tier]]) are
  calibrated on canonical horizons (7/30/90/365 days). A custom-window UAD total may
  be compared with the nearest canonical bucket but must not be mapped onto tier
  labels.
- **Session totals can diverge a few percent from the raw session tables** (~2.4%
  observed on the reference instance): the cube build applies stricter
  database-population and name-resolution filters in the ETL than the raw daily
  views do. Each source is internally consistent — do not mix the two in one
  comparison and read the difference as a change.

## Sources

ETL view definition (`cqx_data` initialization) over `dda_dm.dbusage_cube_activity_repset`,
filter: `replicaid IS NOT NULL AND week_name IS NOT NULL`, all other dimensions NULL.
Additivity verified empirically 2026-07-27 (weekly sums equal all-time totals exactly).
