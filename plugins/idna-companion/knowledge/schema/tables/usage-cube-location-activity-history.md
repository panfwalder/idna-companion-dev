---
id: schema/tables/usage-cube-location-activity-history
title: cqx_data.usage_cube_location_activity_history
type: schema
status: verified
tags: [domain/idna, topic/usage, topic/organization]
related:
  - schema/tables/location
  - schema/tables/usage-cube-location-summary
  - schema/tables/usage-cube-department-activity-history
  - schema/topics/department-and-location
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md
last_reviewed: 2026-04-17
---

# cqx_data.usage_cube_location_activity_history

## Purpose

Weekly activity history by location, based on the usage cube and excluding summary rows. Use this for trend questions such as "how has location X's usage changed over time?"

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `location_id` | bigint | PK/FK to `location`. |
| `location_name` | text | Denormalized location name. |
| `week_name` | text | Human-readable week label. |
| `week_date` | date | Week anchor date. |
| `repsets_accessed` | integer | Count of replica sets accessed that week. |
| `departments_active` | integer | Count of contributing departments that week. |
| `users_active` | integer | Distinct active users that week. |
| `days_active` | integer | Distinct active days that week. |
| `user_access_days` | integer | UAD that week. |
| `sessions_total` | bigint | Total sessions that week. |
| `sessions_client` | bigint | Notes-client sessions that week. |
| `sessions_web` | bigint | Web sessions that week. |
| `reads_client` | bigint | Client-side reads that week. |
| `writes_client` | bigint | Client-side writes that week. |

## Relationships

- PK/FK: `location_id` -> `cqx_data.location.location_id`.

## Typical use

```sql
SELECT week_date, user_access_days, users_active, repsets_accessed
FROM cqx_data.usage_cube_location_activity_history
WHERE location_id = <location_id>
ORDER BY week_date DESC
```

## Notes and caveats

- This is a weekly trend view, not raw sessions.
- Below-threshold locations appear with pseudonymized (masked) names per iDNA's ETL threshold rule; the rows and counts stay present. See [[schema/topics/department-and-location]].

## Sources

CQX schema reference.
