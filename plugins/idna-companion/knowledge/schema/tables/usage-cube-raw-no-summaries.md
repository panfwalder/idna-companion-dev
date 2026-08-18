---
id: schema/tables/usage-cube-raw-no-summaries
title: cqx_data.usage_cube_raw_no_summaries
type: schema
status: verified
tags: [domain/idna, topic/usage]
related:
  - schema/tables/usage-cube-raw
  - schema/tables/usage-sessions
  - schema/tables/usage-cube-repset-vs-user
  - concepts/regulatory-usage-constraints
requires_capability: mcp_unrestricted
source: ETL view definition (cqx_data_ifa), verified 2026-07-27
last_reviewed: 2026-07-27
---

# cqx_data.usage_cube_raw_no_summaries

## Purpose

The **leaf grain** of the usage cube: one row per (replica set × user × department ×
location × ISO week), all five dimensions guaranteed `NOT NULL`. The safe raw slice —
no grouping-set trap, no subtotal rows. Use it when you need per-user-per-week rows
and the metrics it carries are sufficient.

**Deny-listed in restricted mode** (carries `username`). Privileged profile only.

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `replicaid` | text | Application (replica set). |
| `database_title` | text | Application title (denormalized). |
| `username` | text | Notes user name (or `*** Anonymous ***`). |
| `location_name` | text | Location **name only — no `location_id`**. |
| `department_name` | text | Department **name only — no `department_id`**. |
| `week_name` | text | ISO week label. |
| `week_date` | date | Week anchor date (Monday). |
| `sessions_total` | bigint | Sessions that week. |
| `sessions_client` | bigint | Notes-client sessions. |
| `sessions_web` | bigint | Web sessions. |
| `reads_client` | bigint | Client-side reads. |
| `writes_client` | bigint | Client-side writes. |

## What it does NOT carry

- **No `user_access_days`, no `days_active`, no `users_active`** — iDNA's canonical
  usage metric is absent at this grain. For per-user weekly UAD, fall back to
  [[schema/tables/usage-cube-raw]] with all five dimensions pinned `NOT NULL`.
- **No last-access columns.**
- **No `department_id` / `location_id`** — names only. Name-only grouping interacts
  badly with pseudonymized (`*** Obfuscated ... ***`) and `*** Unknown ... ***`
  names: distinct org units can collapse into one masked label. Join-based analysis
  should go through `usage_cube_raw` (IDs present) instead.

## Relationships

- Leaf slice of [[schema/tables/usage-cube-raw]] (`WHERE` all five dims `NOT NULL`).
- Weekly, not daily — per-user *daily* analysis needs
  [[schema/tables/usage-sessions]] plus the person resolution described there.

## Typical use

Per-user weekly session/read/write series for one application:

```sql
SELECT username, week_date, sessions_total, reads_client, writes_client
FROM cqx_data.usage_cube_raw_no_summaries
WHERE replicaid = '<replicaid>'
  AND week_date >= '<from_date>'
ORDER BY week_date, sessions_total DESC
```

## Notes and caveats

- Safe to aggregate freely — every row is a disjoint leaf cell.
- `*** Anonymous ***` rows are real unattributable activity; keep them visible in
  totals rather than filtering them out.
- Eval/trial license masks names (titles, usernames, org names; since the M9 ETL revision, fully trial-selected replica sets show their real title) while structure
  remains.

## Sources

ETL view definition (`cqx_data` initialization): leaf filter over
`dda_dm.dbusage_cube_activity_repset`, projecting the session metrics only.
