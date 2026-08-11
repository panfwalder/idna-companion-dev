---
id: schema/tables/database-content-retention
title: cqx_data.database_content_retention
type: schema
status: verified
tags: [domain/idna, topic/content-age, audience/consultant]
related:
  - schema/tables/database-content-retention-summary
  - schema/tables/database
  - playbooks/application-lifecycle/sunset-funnel
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md
last_reviewed: 2026-05-04
---

# cqx_data.database_content_retention

## Purpose

Monthly distribution of document modifications per scanned NSF, with a flag for whether each monthly bucket sits inside the configured retention window. One row per `database_id` x `last_modification_monthly` bucket. Use this table only when an answer needs the **monthly distribution around the retention cutoff** for evidence narratives — e.g., "how much of this app's content modification falls inside the retention window vs just before it."

For everything else, use [[schema/tables/database-content-retention-summary]]. The summary already exposes in/out-of-retention counts, retention configuration, and recency at both DB and replica-set grain, so the detail table is rarely the right first stop.

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `database_id` | bigint | PK component / FK to `cqx_data.database`. |
| `last_modification_monthly` | date | Calendar month bucket (first day of month). |
| `documents_modified` | integer | Documents whose latest modification falls in this month bucket. |
| `inside_retention_period` | boolean | True if the bucket sits inside the configured retention window. |
| `retention_period_minimum` | date | Boundary date for the configured retention period at scan time. |
| `retention_period_years` | integer | Configured retention period (years). |
| `retention_mode` | text | Retention configuration mode. NULL when not configured. |

## Relationships

- PK: `database_id` (the table effectively keys on `database_id` + `last_modification_monthly`; the schema reference declares `database_id` as PK).
- `database_id` -> `cqx_data.database.database_id`. There is **no `replicaid`** on this table; aggregate to replica-set grain by joining `cqx_data.database` first.

## Typical use

Intent: monthly distribution of document modifications for one database, with retention-window flag, around the retention cutoff.

```sql
SELECT last_modification_monthly,
  documents_modified,
  inside_retention_period
FROM cqx_data.database_content_retention
WHERE database_id = <database_id>
  AND retention_period_minimum IS NOT NULL
ORDER BY last_modification_monthly DESC
```

## Notes and caveats

- **Use the summary first.** If the question is "is this app retention-locked," the summary's `docs_count_inret` already answers it; pulling this detail page in addition wastes context.
- No `replicaid`. To aggregate across a replica set, join `cqx_data.database` and group by its `replicaid`.
- **NULL `retention_mode` means retention has not been configured for the parent DB.** NULL `retention_period_minimum` covers a broader case: both unconfigured *and* configured-but-incomplete (`retention_mode` set without `retention_period_years`, so ETL never produced a boundary date). Filter `retention_period_minimum IS NOT NULL` for retention-aware queries; rows without a boundary date cannot be interpreted.
- Bucket boundary is calendar-month; document-level dating beyond month granularity requires `cqx_data.design_element` or other document-level tables, not this view.

## Sources

CQX schema reference.
