---
id: schema/tables/usage-cube-location-summary
title: cqx_data.usage_cube_location_summary
type: schema
status: verified
tags: [domain/idna, topic/usage, topic/organization]
related:
  - schema/tables/location
  - schema/tables/usage-cube-location-activity-history
  - schema/tables/usage-cube-department-summary
  - schema/topics/department-and-location
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md
last_reviewed: 2026-04-17
---

# cqx_data.usage_cube_location_summary

## Purpose

Per-location summary of the usage cube across all applications, excluding summary rows. Use this for overall activity by location rather than app-specific analysis.

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `location_id` | bigint | PK and join key to `location`. |
| `location_name` | text | Denormalized location name. |
| `last_access` | date | Most recent read across all apps for this location. |
| `last_access_category` | text | Bucketed recency label. |
| `days_since_last_access` | integer | Days since `last_access`. |
| `last_write_access` | date | Most recent write across all apps for this location. |
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
| `departments_active` | integer | Count of contributing departments. |

## Relationships

- PK/FK: `location_id` -> `cqx_data.location.location_id`.

## Typical use

```sql
SELECT location_name, user_access_days, users_active, repsets_accessed, last_access
FROM cqx_data.usage_cube_location_summary
ORDER BY user_access_days DESC NULLS LAST
LIMIT 20
```

## Notes and caveats

- This is an aggregate across all applications; use `usage_cube_repset_vs_location` when the question is about one application.
- Below-threshold locations appear with pseudonymized (masked) names per iDNA's ETL threshold rule; the rows and counts stay present. See [[schema/topics/department-and-location]].

## Sources

CQX schema reference.
