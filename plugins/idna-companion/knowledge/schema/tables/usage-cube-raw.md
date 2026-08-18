---
id: schema/tables/usage-cube-raw
title: cqx_data.usage_cube_raw
type: schema
status: verified
tags: [domain/idna, topic/usage]
related:
  - schema/tables/usage-cube-raw-no-summaries
  - schema/tables/usage-cube-repset-vs-user
  - schema/tables/usage-cube-repset-activity-history
  - schema/tables/usage-cube-weekly-summary
  - concepts/user-access-days
  - concepts/regulatory-usage-constraints
requires_capability: mcp_unrestricted
source: ETL cube build SQL + live grouping-set census 2026-07-27
last_reviewed: 2026-07-27
---

# cqx_data.usage_cube_raw

## Purpose

The **unfiltered** usage cube: every grouping-set row of
`dda_dm.dbusage_cube_activity_repset` — leaf rows *and* all subtotal rows. This view
exists so consumers can pick any slice of the (replica set × user × department ×
location × week) cube; it is **not** a fact table, and treating it as one is the
single most dangerous mistake in the usage schema.

**Deny-listed in restricted mode** (carries `username`). Privileged profile only.

## The trap, stated first

The cube is built with `GROUP BY CUBE(replicaid, username, dep_id, loc_id, week_name)`
and a redundancy filter, leaving **20 distinct NULL-ness patterns** in this view
(the 12 username-bearing subtotal patterns that roll up department or location are
pruned, because a user has exactly one of each; username-bearing rows therefore
always carry both `department_id` and `location_id`).

`NULL` in a dimension column means **"rolled up over this dimension"** — not missing
data. Any aggregate over this view without grouping-aware filters counts every
activity multiple times (once per matching grouping set). **Always pin every
dimension** to either `IS NULL` (rolled up) or `IS NOT NULL` (broken out) — never
leave one unconstrained.

## Key columns

All columns of the underlying cube: the five dimensions (`replicaid` +
`database_title`, `username`, `department_id` + `department_name`, `location_id` +
`location_name`, `week_name` + `week_date`), recency columns (`last_access`,
`last_access_category`, `days_since_last_access`, and the `_write_` variants), and
metrics (`sessions_total/client/web`, `reads_client`, `writes_client`,
`user_access_days`, `users_active`, `days_active`, `repsets_accessed`,
`departments_active`, `locations_active`).

Sentinel values (leaf rows): `department_id`/`location_id` = **`-1`** means
*unknown/unmapped* (a real cohort — do not filter it away); `username` =
`*** Anonymous ***` for unattributable sessions. `NULL` always means *rolled up*,
never *unknown*.

## Relationships

- Every named `cqx_data.usage_cube_*` view is a grouping-set slice of this cube —
  prefer those views; they encode the dimension filters correctly. E.g.
  [[schema/tables/usage-cube-repset-activity-history]] = (replicaid, week) slice,
  [[schema/tables/usage-cube-weekly-summary]] = (week) slice,
  [[schema/tables/usage-cube-repset-vs-user]] = (replicaid, username) slice.
- Use this view directly only when no named slice fits — chiefly the per-user-per-week
  leaf **with UAD**, which [[schema/tables/usage-cube-raw-no-summaries]] lacks.

## Typical use

Per-user weekly activity for one application, with UAD (the slice no named view
covers):

```sql
SELECT username, week_date, user_access_days, sessions_total, reads_client, writes_client
FROM cqx_data.usage_cube_raw
WHERE replicaid = '<replicaid>'      -- pinned: broken out
  AND username IS NOT NULL           -- pinned: broken out
  AND week_name IS NOT NULL          -- pinned: broken out
  AND department_id IS NOT NULL      -- pinned (always set on username rows)
  AND location_id IS NOT NULL        -- pinned (always set on username rows)
ORDER BY week_date, user_access_days DESC
```

## Notes and caveats

- **Never aggregate without pinning all five dimensions** (see trap above).
- Do not query one of the 12 pruned grouping sets (username broken out, dep or loc
  rolled up) — it returns zero rows, which looks like "no data" but means "this
  subtotal does not exist"; use the dep/loc-pinned form instead.
- Eval/trial license masks *names* here (`*** Trial license ***` patterns; since the M9 ETL revision, fully trial-selected replica sets show their real title instead) while IDs
  remain populated — the opposite of `user_activity_summary`, which NULLs the IDs.
- Weekly grain: per-user *daily* analysis needs `usage_sessions`, not the cube.

## Sources

Cube build (`320_dda_dm.dbusage_cube_activity_repset.update.sql`): `GROUP BY CUBE`
plus redundancy filter `WHERE username IS NULL OR (dep_id IS NOT NULL AND loc_id IS
NOT NULL)`. Grouping-set census verified live 2026-07-27: exactly 20 patterns.
