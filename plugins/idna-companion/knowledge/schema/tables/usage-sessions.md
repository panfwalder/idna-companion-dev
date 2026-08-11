---
id: schema/tables/usage-sessions
title: cqx_data.usage_sessions
type: schema
status: verified
tags: [domain/idna, topic/usage]
related:
  - concepts/user-access-days
  - concepts/regulatory-usage-constraints
  - schema/tables/database
  - schema/tables/usage-cube-repset-vs-user
  - schema/tables/usage-sessions-by-department
  - schema/tables/usage-sessions-by-location
  - howtos/identify-content-creators-and-consumers
requires_capability: mcp_unrestricted
source: docs/reference/cqx_data_schema_enriched_all_tables.md
last_reviewed: 2026-07-27
---

# cqx_data.usage_sessions

## Purpose

Raw session detail. One row per (database, day, user, server) with byte counters, time-open, and channel split (Notes client vs web vs web mobile). The most granular per-user usage table in the wiki — drop into this only when the per-user cube and time-period summary cannot answer the question. **Deny-listed in restricted mode** per [[concepts/regulatory-usage-constraints]].

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `session_date` | date | Day the sessions occurred. |
| `person_id` | bigint | Join key to `person`. Identifies the user. |
| `database_id` | bigint | Join key to `database`. The instance accessed (not the replica set). |
| `server_id` | bigint | Join key to `server`. The Domino server the session ran on. |
| `access_type` | text | Channel/access category label. |
| `sessions_client` | integer | Notes-client sessions on the day. |
| `reads_client` | integer | Client-side reads. |
| `writes_client` | integer | Client-side writes. |
| `transactions_client` | bigint | Client-side transactions. |
| `bytestoclient_client` | bigint | Bytes sent to the Notes client. |
| `bytestoserver_client` | bigint | Bytes received from the Notes client. |
| `timeopen_client` | interval | Total time the database was open in the Notes client. |
| `sessions_web` | integer | Browser/web sessions on the day. |
| `sessions_web_mobile` | integer | Sessions identified as mobile-web. |
| `bytestoclient_web` | bigint | Bytes sent to the browser. |
| `bytestoserver_web` | bigint | Bytes received from the browser. |
| `mails_sent_web` | smallint | Mails sent through the web client. |
| `mails_sent_attachments_web` | smallint | Mails sent through the web client that carried attachments. |

## Relationships

- `person_id` -> `cqx_data.person.person_id` (note: `person` itself is also deny-listed).
- `database_id` -> `cqx_data.database.database_id`.
- `server_id` -> `cqx_data.server.server_id`.
- No FK to `replica_set` — `usage_sessions` is per-instance, not per-application. Aggregate to replica-set grain via `database` -> `replicaid`.

## Typical use

Raw session granularity is rarely the right tool — prefer [[schema/tables/usage-cube-repset-vs-user]] for per-user detail or one of the aggregate-grain `usage_sessions_by_*` cousins for org-attribution. The illustrative pattern when this level of detail is required:

```sql
SELECT s.session_date,
  s.access_type,
  s.sessions_client,
  s.reads_client,
  s.writes_client,
  s.timeopen_client
FROM cqx_data.usage_sessions s
WHERE s.person_id = <person_id>
  AND s.database_id = <database_id>
ORDER BY s.session_date DESC
LIMIT 100
```

Wrapping recipe (where this granularity is occasionally cited as a fallback): [[howtos/identify-content-creators-and-consumers]].

## Notes and caveats

- **Deny-listed in restricted mode.** Same posture as [[schema/tables/usage-cube-repset-vs-user]] and [[schema/tables/repset-activity-summary-per-user]] — see [[concepts/regulatory-usage-constraints]].
- **`person_id` can be NULL — anonymous sessions are real activity.** Sessions with no resolvable node ID (predominantly anonymous web access; ~20% of web session volume on the reference instance, verified 2026-07-27) carry `person_id IS NULL`. Any INNER JOIN on `person_id` silently drops them — use LEFT JOINs and surface the unattributable share explicitly rather than losing it.
- Keyed on `person_id` rather than `username`. The pseudonymization posture applies through `person` (also deny-listed); under `pseudonymize=Y` the join produces hashed names, under `pseudonymize=N` real names.
- Aggregate-grain twins that *are* allowlisted under rev 6.3: [[schema/tables/usage-sessions-by-department]], [[schema/tables/usage-sessions-by-location]]. Prefer those for org-attribution questions.
- The mail counters (`mails_sent_web`, `mails_sent_attachments_web`) are present but per-user mail analysis lives more naturally on the `mail_*` family. This page does not cover that surface.

## Sources

CQX schema reference.
