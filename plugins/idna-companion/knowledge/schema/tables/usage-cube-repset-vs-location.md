---
id: schema/tables/usage-cube-repset-vs-location
title: cqx_data.usage_cube_repset_vs_location
type: schema
status: verified
tags: [domain/idna, topic/usage, topic/organization]
related:
  - concepts/user-access-days
  - schema/tables/replica-set
  - schema/tables/location
  - schema/tables/usage-cube-repset-vs-department
  - schema/tables/usage-cube-repset-vs-user
  - schema/topics/department-and-location
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md
last_reviewed: 2026-05-08
---

# cqx_data.usage_cube_repset_vs_location

## Purpose

Application x location usage cross-reference. Answers "which locations use application X" and "what applications does location Y use." This is the location twin of [[schema/tables/usage-cube-repset-vs-department]].

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `replicaid` | text | Join key to `replica_set`. |
| `database_title` | text | Application title (denormalized). |
| `location_id` | bigint | Join key to `location`. |
| `location_name` | text | Denormalized location name. |
| `last_access` | date | Most recent read by this location. |
| `last_access_category` | text | Bucketed recency label. |
| `days_since_last_access` | integer | Days since `last_access`. |
| `last_write_access` | date | Most recent write by this location. |
| `last_write_access_category` | text | Bucketed recency of last write. |
| `days_since_last_write_access` | integer | Days since `last_write_access`. |
| `locations_active` | integer | Distinct locations contributing usage. |
| `users_active` | integer | Distinct users in this location who accessed. |
| `days_active` | integer | Distinct days with access. |
| `user_access_days` | integer | UAD for this (application, location) pair. |
| `sessions_total` | bigint | Total sessions. |
| `sessions_client` | bigint | Sessions from Notes client. |
| `sessions_web` | bigint | Sessions via browser/web channel. |
| `reads_client` | bigint | Client-side reads. |
| `writes_client` | bigint | Client-side writes. |

## Relationships

- `location_id` -> `cqx_data.location.location_id`.
- `replicaid` -> `cqx_data.replica_set.replicaid`.
- The schema reference lists `location_id` as the PK for this view; in practice treat the useful grain as `(replicaid, location_id)`.

## Typical use

```sql
SELECT rs.title, l.location_name, l.user_access_days, l.users_active, l.last_access
FROM cqx_data.replica_set rs
JOIN cqx_data.usage_cube_repset_vs_location l USING (replicaid)
WHERE rs.replicaid = '<replicaid>'
ORDER BY l.user_access_days DESC NULLS LAST
```

## Notes and caveats

- Below-threshold locations appear with pseudonymized (masked) names per iDNA's ETL threshold rule; the rows and counts stay present. See [[schema/topics/department-and-location]].
- For time trends use `usage_cube_location_activity_history`; for per-location summaries across all apps use `usage_cube_location_summary`.

## Sources

CQX schema reference.
