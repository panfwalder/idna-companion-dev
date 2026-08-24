---
id: schema/tables/usage-cube-weekly-summary
title: cqx_data.usage_cube_weekly_summary
type: schema
status: verified
tags: [domain/idna, topic/usage]
related:
  - schema/tables/usage-cube-repset-activity-history
  - schema/tables/usage-cube-department-activity-history
  - schema/tables/usage-cube-location-activity-history
  - concepts/user-access-days
requires_capability: none
source: ETL view definition (cqx_data_ifa) + live verification 2026-07-27
last_reviewed: 2026-08-24
---

# cqx_data.usage_cube_weekly_summary

## Purpose

Estate-level weekly usage summary — one row per ISO week, all applications, all
users, all org units rolled up. The estate-grain counterpart to
[[schema/tables/usage-cube-repset-activity-history]]. Use for "how is overall Notes
usage developing?" and as the estate baseline when a single application's trend needs
context.

Available in **both** access profiles (restricted allowlist).

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `week_name` | text | ISO week label (`ISOYEAR-Wnn`). |
| `week_date` | date | Week anchor date (Monday of the ISO week). |
| `sessions_total` | bigint | Total sessions that week. Additive. |
| `sessions_client` | bigint | Notes-client sessions. Additive. |
| `sessions_web` | bigint | Web sessions. Additive. |
| `reads_client` | bigint | Client-side reads. Additive. |
| `writes_client` | bigint | Client-side writes. Additive. |
| `user_access_days` | integer | Estate UAD that week. Additive. |
| `users_active` | integer | Distinct active users that week. **Not additive across weeks.** |
| `days_active` | integer | Distinct active days that week. |
| `repsets_accessed` | integer | Distinct replica sets accessed that week. **Not additive across weeks.** |
| `departments_active` | integer | Distinct departments active that week. **Not additive.** |
| `locations_active` | integer | Distinct locations active that week. **Not additive.** |

## Relationships

- No join keys — this is the fully rolled-up grain of the usage cube (all
  dimensions NULL except week).
- Per-entity weekly twins: [[schema/tables/usage-cube-repset-activity-history]]
  (per app), [[schema/tables/usage-cube-department-activity-history]] (per
  department), [[schema/tables/usage-cube-location-activity-history]] (per location).

## Typical use

Estate activity over a custom window:

```sql
SELECT
  sum(user_access_days) AS uad,
  sum(sessions_total)   AS sessions,
  max(users_active)     AS users_active_peak_week,
  count(*)              AS weeks_covered
FROM cqx_data.usage_cube_weekly_summary
WHERE week_date >= ('<from_date>'::date - 6) AND week_date <= '<to_date>'::date
```

## Notes and caveats

- The distinct-count columns (`users_active`, `repsets_accessed`,
  `departments_active`, `locations_active`) are per-week distincts — **never sum
  them across weeks**; report peak week or a weekly series instead.
- Whole-week widening (custom windows cover every overlapping Monday-anchored
  week; see `covered_from_date` / `covered_to_date`) and instance-specific
  history depth apply exactly as on
  [[schema/tables/usage-cube-repset-activity-history]] — as does the caveat there
  that cube session totals can diverge a few percent from the raw session tables
  (different ETL population filters; don't mix the two sources in one comparison).

## Sources

ETL view definition (`cqx_data` initialization) over
`dda_dm.dbusage_cube_activity_repset`, filter: `week_name IS NOT NULL`, all other
dimensions NULL.
