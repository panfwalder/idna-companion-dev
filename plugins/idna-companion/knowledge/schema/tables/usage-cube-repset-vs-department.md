---
id: schema/tables/usage-cube-repset-vs-department
title: cqx_data.usage_cube_repset_vs_department
type: schema
status: verified
tags: [domain/idna, topic/usage, topic/organization]
related:
  - concepts/user-access-days
  - schema/tables/replica-set
  - schema/tables/department
  - schema/tables/usage-cube-repset-vs-location
  - schema/tables/usage-cube-repset-vs-user
  - schema/topics/department-and-location
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md
last_reviewed: 2026-05-08
---

# cqx_data.usage_cube_repset_vs_department

## Purpose

Application x department usage cross-reference. Answers "which departments use application X" and "what applications does department Y use." The location twin lives at [[schema/tables/usage-cube-repset-vs-location]].

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `replicaid` | text | Join key to `replica_set`. |
| `database_title` | text | Application title (denormalized). |
| `department_id` | bigint | Join key to `department`. |
| `department_name` | text | Denormalized department name. |
| `last_access` | date | Most recent read by this department. |
| `last_access_category` | text | Bucketed recency label. |
| `days_since_last_access` | integer | Days since `last_access`. |
| `last_write_access` | date | Most recent write by this department. |
| `last_write_access_category` | text | Bucketed recency of last write. |
| `days_since_last_write_access` | integer | Days since `last_write_access`. |
| `locations_active` | integer | Distinct locations contributing usage. |
| `users_active` | integer | Distinct users in this department who accessed. |
| `days_active` | integer | Distinct days with access. |
| `user_access_days` | integer | UAD for this (application, department) pair. |
| `sessions_total` | bigint | Total sessions. |
| `sessions_client` | bigint | Sessions from Notes client. |
| `sessions_web` | bigint | Sessions via browser/web channel. |
| `reads_client` | bigint | Client-side reads. |
| `writes_client` | bigint | Client-side writes. |

## Relationships

- `replicaid` -> `cqx_data.replica_set.replicaid`.
- `department_id` -> `cqx_data.department.department_id`.
- The schema reference lists `department_id` as the PK for this view; in practice treat the useful grain as `(replicaid, department_id)`.

## Typical use

Departments using a given application, ranked by UAD:
```sql
SELECT rs.title, d.department_name, d.user_access_days, d.users_active, d.last_access
FROM cqx_data.replica_set rs
JOIN cqx_data.usage_cube_repset_vs_department d USING (replicaid)
WHERE rs.replicaid = '<replicaid>'
ORDER BY d.user_access_days DESC NULLS LAST
```

## Notes and caveats

- Below-threshold departments appear with pseudonymized (masked) names per iDNA's ETL threshold rule; the rows and counts stay present. See [[schema/topics/department-and-location]].
- For time trends use `usage_cube_department_activity_history`; for per-department summaries across all apps use `usage_cube_department_summary`.

## Sources

CQX schema reference.
