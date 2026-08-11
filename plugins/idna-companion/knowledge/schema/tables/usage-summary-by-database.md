---
id: schema/tables/usage-summary-by-database
title: cqx_data.usage_summary_by_database
type: schema
status: verified
tags: [domain/idna, topic/usage]
related:
  - concepts/database-instance
  - concepts/user-access-days
  - schema/topics/cqx-data-execution-surface
  - schema/tables/database
  - schema/tables/replica-set
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md
last_reviewed: 2026-04-17
---

# cqx_data.usage_summary_by_database

## Purpose

Per-database-instance usage aggregates across 7/30/90/365-day and all-time windows. Use when the question is about a specific server, specific NSF, or per-instance comparison. For application-level usage, prefer the pre-aggregated columns on [[schema/tables/replica-set]].

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `database_id` | integer | Join key to `cqx_data.database`. |
| `title` | text | Database title copied into the summary view. |
| `server` | text | Server name copied into the summary view. |
| `filename` | text | File path copied into the summary view. |
| `is_focus_db` | boolean | Focus flag for the instance. |
| `database_type` | text | Database-type label for the instance. |
| `user_access_days_last7d` | integer | UAD in the last 7 days. |
| `user_access_days_last30d` | integer | UAD in the last 30 days. |
| `user_access_days_last90d` | integer | UAD in the last 90 days. |
| `user_access_days_last365d` | integer | UAD in the last 365 days. |
| `user_access_days_all_time` | integer | UAD across all recorded history. |
| `users_active_last7d` | integer | Distinct user count in the last 7 days. |
| `users_active_last30d` | integer | Distinct user count in the last 30 days. |
| `users_active_last90d` | integer | Distinct user count in the last 90 days. |
| `sessions_last90d` | bigint | Session count in the last 90 days. |
| `sessions_web_last90d` | bigint | Web session count in the last 90 days. |
| `sessions_write_last90d` | bigint | Write-session count in the last 90 days. |
| `last_accessed` | date | Most recent read. |
| `last_accessed_user` | text | Last user recorded as reader. |
| `last_write_access` | date | Most recent write. |
| `last_write_user` | text | Last user recorded as writer. |

## Relationships

- `database_id` -> `cqx_data.database.database_id`.
- To aggregate to application level, join through `cqx_data.database.replicaid` to `cqx_data.replica_set`.

## Typical use

```sql
SELECT d.title, d.server_name, d.template_inherits_from,
  u.user_access_days_last90d, u.users_active_last90d, u.last_accessed
FROM cqx_data.database d
JOIN cqx_data.usage_summary_by_database u ON u.database_id = d.database_id
WHERE d.is_focus_db
ORDER BY u.user_access_days_last90d DESC NULLS LAST
LIMIT 20
```

## Notes and caveats

- This view already carries denormalized title/server/file metadata, so many quick reports can use it directly without joining back to `database`.
- Per-instance usage can differ wildly across a replica set; one server may hold the primary replica while others are rarely opened.
- NULL UAD means no recorded usage; order with `NULLS LAST`.
- Architectural context: because `cqx_data` is the public execution surface, this is a better first stop for consultant-facing SQL than tracing directly into upstream mart tables. See [[schema/topics/cqx-data-execution-surface]].

## Sources

CQX schema reference.
