---
id: schema/tables/repset-activity-summary-per-user
title: cqx_data.repset_activity_summary_per_user
type: schema
status: verified
tags: [domain/idna, topic/usage]
related:
  - concepts/user-access-days
  - concepts/regulatory-usage-constraints
  - schema/tables/replica-set
  - schema/tables/usage-cube-repset-vs-user
  - howtos/identify-content-creators-and-consumers
requires_capability: mcp_unrestricted
source: docs/reference/cqx_data_schema_enriched_all_tables.md
last_reviewed: 2026-05-08
---

# cqx_data.repset_activity_summary_per_user

## Purpose

Per-user activity summary for one replica set, split across multiple time periods (30 days, 90 days, all recorded history). One row per (replica set, user, time period). Use when the question requires comparing a user's activity across windows; for the richer single-window view (read/write split, sessions breakdown) prefer [[schema/tables/usage-cube-repset-vs-user]]. **Deny-listed in restricted mode** per [[concepts/regulatory-usage-constraints]].

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `replicaid` | text | Join key to `replica_set`. |
| `username` | text | Notes user name (real or pseudonymized depending on the customer's `pseudonymize` flag). |
| `last_access` | date | Most recent read by this user within the time period. |
| `last_write_access` | date | Most recent write by this user within the time period. |
| `sessions_total` | bigint | Total sessions in the time period. |
| `sessions_write` | bigint | Sessions in the time period that included writes. |
| `time_period` | text | The window label (e.g. `30d`, `90d`, `all_time`). |

## Relationships

- `replicaid` -> `cqx_data.replica_set.replicaid`.
- The useful query grain is `(replicaid, username, time_period)`.

## Typical use

User activity for one application across the standard time-period windows:

```sql
SELECT a.username, a.time_period,
  a.sessions_total, a.sessions_write,
  a.last_access, a.last_write_access
FROM cqx_data.repset_activity_summary_per_user a
WHERE a.replicaid = '<replicaid>'
ORDER BY a.username, a.time_period
```

Wrapping recipe: [[howtos/identify-content-creators-and-consumers]] (Variations section).

## Notes and caveats

- **Deny-listed in restricted mode.** Same posture as [[schema/tables/usage-cube-repset-vs-user]] — see [[concepts/regulatory-usage-constraints]] for the framing.
- Time period values are determined by ETL configuration; treat them as opaque strings unless you control the ETL.
- The richer per-user surface (read/write split, sessions_client / sessions_web, days_active) lives on [[schema/tables/usage-cube-repset-vs-user]]. This table is the time-period-aware companion.
- `username` posture identical to `usage-cube-repset-vs-user`: hashed when `pseudonymize=Y`, real when `pseudonymize=N`.

## Sources

CQX schema reference.
