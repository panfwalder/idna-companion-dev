---
id: schema/tables/usage-cube-department-summary
title: cqx_data.usage_cube_department_summary
type: schema
status: verified
tags: [domain/idna, topic/usage, topic/organization]
related:
  - schema/tables/department
  - schema/tables/usage-cube-department-activity-history
  - schema/tables/usage-cube-location-summary
  - schema/topics/department-and-location
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md
last_reviewed: 2026-04-17
---

# cqx_data.usage_cube_department_summary

## Purpose

Per-department summary of the usage cube across all applications, excluding summary rows. Use this for "what is the overall activity profile of department X?" rather than for app-specific questions.

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `department_id` | bigint | PK and join key to `department`. |
| `department_name` | text | Denormalized department name. |
| `last_access` | date | Most recent read across all apps for this department. |
| `last_access_category` | text | Bucketed recency label. |
| `days_since_last_access` | integer | Days since `last_access`. |
| `last_write_access` | date | Most recent write across all apps for this department. |
| `last_write_access_category` | text | Bucketed recency of last write. |
| `days_since_last_write_access` | integer | Days since `last_write_access`. |
| `sessions_total` | bigint | Total sessions. |
| `sessions_client` | bigint | Notes-client sessions. |
| `sessions_web` | bigint | Web sessions. |
| `reads_client` | bigint | Client-side reads. |
| `writes_client` | bigint | Client-side writes. |
| `user_access_days` | integer | UAD across all applications. |
| `users_active` | integer | Distinct active users. |
| `days_active` | integer | Distinct active days. |
| `repsets_accessed` | integer | Count of replica sets accessed. |
| `locations_active` | integer | Count of contributing locations. |

## Relationships

- PK/FK: `department_id` -> `cqx_data.department.department_id`.

## Typical use

```sql
SELECT department_name, user_access_days, users_active, repsets_accessed, last_access
FROM cqx_data.usage_cube_department_summary
ORDER BY user_access_days DESC NULLS LAST
LIMIT 20
```

## Notes and caveats

- This is an org-level aggregate across all applications; use `usage_cube_repset_vs_department` when the question is scoped to one application.
- Below-threshold departments appear with pseudonymized (masked) names per iDNA's ETL threshold rule; the rows and counts stay present. See [[schema/topics/department-and-location]].

## Sources

CQX schema reference.
