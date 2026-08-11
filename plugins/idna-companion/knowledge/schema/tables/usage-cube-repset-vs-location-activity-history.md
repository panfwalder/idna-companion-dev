---
id: schema/tables/usage-cube-repset-vs-location-activity-history
title: cqx_data.usage_cube_repset_vs_location_activity_history
type: schema
status: verified
tags: [domain/idna, topic/usage, topic/organization]
related:
  - schema/tables/usage-cube-repset-vs-department-activity-history
  - schema/tables/usage-cube-repset-vs-location
  - schema/tables/usage-cube-repset-activity-history
  - schema/tables/usage-cube-location-activity-history
  - schema/tables/location
  - schema/topics/department-and-location
  - concepts/usage-time-windows
requires_capability: none
source: ETL view definition (cqx_data_ifa, added 2026-07-28) + live validation
last_reviewed: 2026-07-28
---

# cqx_data.usage_cube_repset_vs_location_activity_history

## Purpose

Weekly activity history at **application × location** grain — the location twin
of [[schema/tables/usage-cube-repset-vs-department-activity-history]] and the
windowed counterpart of [[schema/tables/usage-cube-repset-vs-location]]. One row
per (replicaid, location_id, ISO week). Backs `usage_over_window` cross mode
for location questions ("is app X used more from Vienna or Berlin this year?").

Available in **both** access profiles (restricted allowlist, org-attribution
aggregate category).

## Key columns

Identical shape to the department twin, with `location_id` / `location_name` as
the org dimension: `replicaid`, `database_title`, `week_name`, `week_date`,
`users_active` (per-week distinct, **not additive across weeks**),
`days_active`, `user_access_days`, `sessions_total/client/web`, `reads_client`,
`writes_client` (all additive across weeks). `location_id = -1` is the
unattributed cohort.

## Relationships

- `replicaid` -> `cqx_data.replica_set.replicaid`.
- `location_id` -> `cqx_data.location.location_id` (except `-1`).
- Department twin: [[schema/tables/usage-cube-repset-vs-department-activity-history]].

## Typical use

Same query patterns as the department twin with the location columns
substituted; see that page. Subtree questions use
`location.path @> ARRAY[<id>]`.

## Notes and caveats

All caveats of the department twin apply unchanged: leaf grain (no hierarchy
roll-up), `-1` cohort is real activity, peak-week-only `users_active`
(within-week sums across sibling locations are fine), ISO-week floor,
instance-specific depth, threshold-pseudonymized names.

## Sources

ETL view (added 2026-07-28) projecting the (replicaid × location × week)
grouping set of `dda_dm.dbusage_cube_activity_repset`; validated live the same
day.
