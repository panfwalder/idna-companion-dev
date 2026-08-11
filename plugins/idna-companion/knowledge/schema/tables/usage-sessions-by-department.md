---
id: schema/tables/usage-sessions-by-department
title: cqx_data.usage_sessions_by_department
type: schema
status: verified
tags: [domain/idna, topic/usage, topic/organization]
related:
  - schema/tables/database
  - schema/tables/department
  - schema/tables/usage-cube-department-activity-history
  - schema/tables/usage-cube-repset-vs-department-activity-history
  - schema/tables/usage-sessions
  - schema/topics/department-and-location
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md
last_reviewed: 2026-07-28
---

# cqx_data.usage_sessions_by_department

## Purpose

Session-level usage detail per database, day, and department. Use this only when the pre-aggregated cube views do not expose the grain you need.

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `database_id` | bigint | FK to `database`. |
| `server_id` | bigint | FK to `server`. |
| `department_id` | bigint | FK to `department`. |
| `session_date` | date | Session day. |
| `sessions_client` | bigint | Notes-client sessions. |
| `reads_client` | bigint | Client-side reads. |
| `writes_client` | bigint | Client-side writes. |
| `transactions_client` | numeric | Client transaction count. |
| `bytestoclient_client` | numeric | Bytes sent to client. |
| `bytestoserver_client` | numeric | Bytes sent to server. |
| `timeopen_client` | interval | Time-open measure for client sessions. |
| `sessions_web` | bigint | Web sessions. |
| `sessions_web_mobile` | bigint | Web/mobile sessions. |
| `bytestoclient_web` | numeric | Bytes sent to web client. |
| `bytestoserver_web` | numeric | Bytes sent to server via web channel. |
| `mails_sent_web` | bigint | Mail sent through web channel. |
| `mails_sent_attachments_web` | bigint | Web mail with attachments count. |

## Relationships

- `database_id` -> `cqx_data.database.database_id`.
- `department_id` -> `cqx_data.department.department_id`.
- `server_id` -> `cqx_data.server.server_id`.

## Typical use

```sql
SELECT session_date, sessions_client, sessions_web, reads_client, writes_client
FROM cqx_data.usage_sessions_by_department
WHERE department_id = <department_id>
ORDER BY session_date DESC
LIMIT 50
```

## Notes and caveats

- Prefer cube summary/history tables first; this view is for deeper forensics and mail-traffic-style analysis.
- Because this is lower-grain data, result volumes can grow quickly.
- **Performance trap for windowed app×department questions:** rows are hierarchy roll-ups (`unnest(path)` — each department row includes its descendants), and filtering by `department_id` forces a full unnest scan (~90s measured on a small estate, 2026-07-28). For custom-window app×department analysis use [[schema/tables/usage-cube-repset-vs-department-activity-history]] instead — weekly grain, full metric set, fast in both directions.

## Sources

CQX schema reference.
