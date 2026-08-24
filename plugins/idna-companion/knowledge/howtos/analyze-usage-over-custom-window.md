---
id: howtos/analyze-usage-over-custom-window
title: Analyze usage over a custom time window
type: howto
status: verified
tags: [domain/idna, topic/usage, audience/consultant]
related:
  - concepts/usage-time-windows
  - concepts/user-access-days
  - concepts/usage-tier
  - schema/tables/usage-cube-repset-activity-history
  - schema/tables/usage-cube-weekly-summary
  - schema/tables/usage-cube-department-activity-history
  - schema/tables/usage-cube-location-activity-history
  - schema/tables/usage-cube-repset-vs-department-activity-history
  - schema/tables/usage-cube-repset-vs-location-activity-history
  - howtos/attribute-usage-aggregate-only
requires_capability: none
source: user-activity-time-windows plan (2026-07-27); cross-mode rewrite 2026-07-28; rank apps_used amendment 2026-07-29; window-coverage fix (overlap-week predicate) 2026-08-24
last_reviewed: 2026-08-24
---

# Analyze usage over a custom time window

## Question

"How active was app X in the last six months?" / "Which are my most active apps
this year?" / "Which departments have been most active this year in app X?" —
any usage question naming a window that is not one of iDNA's canonical horizons
(7/30/90/365 days, all-time). The framing rules — when to redirect to buckets,
when to answer the window directly — live in [[concepts/usage-time-windows]];
this page is the execution recipe.

## Approach

**The `usage_over_window` MCP tool is the primary surface** — it works in every
access profile, encodes the additivity rules, embeds the nearest canonical
bucket, and reports requested-vs-effective window when history is shorter than
asked. Three modes:

- **series** — `scope` (`estate|app|department|location`) + `scope_id`: weekly
  trend + exact window totals for one entity.
- **rank** — scope without `scope_id`: top-N entities by an additive metric
  over the window; `include_unused: true` inverts to entities with NO recorded
  activity in the window.
- **cross** — one app split by department/location (`by`), or one
  department/location's top apps (`by: app`), at weekly grain with the full
  metric set.

Department/location references are **subtree-inclusive by default**
(`include_subtree: false` for direct members only — series/rank only; the
cross-mode backing tables are pre-rolled-up).

Manual SQL (Metabase, psql) uses the same tables the tool reads; patterns below.

## Data needed

- [[schema/tables/usage-cube-repset-activity-history]] — per-app weekly.
- [[schema/tables/usage-cube-weekly-summary]] — estate weekly.
- [[schema/tables/usage-cube-department-activity-history]] /
  [[schema/tables/usage-cube-location-activity-history]] — per-org weekly.
- [[schema/tables/usage-cube-repset-vs-department-activity-history]] /
  [[schema/tables/usage-cube-repset-vs-location-activity-history]] — app×org
  weekly (full metric set incl. UAD; unattributed cohort as org id `-1`).

## Procedure

### Variant A — one application over a window

`usage_over_window` with `scope: app`, `scope_id: <replicaid>`, `from_date`,
`to_date`. Manual pattern:

```sql
SELECT
  sum(user_access_days) AS uad,
  sum(sessions_total)   AS sessions,
  max(users_active)     AS users_active_peak_week,
  count(*)              AS weeks_covered
FROM cqx_data.usage_cube_repset_activity_history
WHERE replicaid = '<replicaid>'
  AND week_date >= ('<from_date>'::date - 6) AND week_date <= '<to_date>'::date
```

Present alongside the canonical bucket (`usage_last90d` / `usage_last365d` and
the UAD window columns on `cqx_data.replica_set`).

### Variant B — top apps over a window

`usage_over_window` with `scope: app`, no `scope_id` (rank mode). Manual: same
table, `GROUP BY replicaid`, `ORDER BY sum(user_access_days) DESC`. Rank mode
also answers the org-wide matrix question — "how many apps were used by which
departments in July?" — in one call, both profiles: `scope: department` (or
`location`) with `metric: apps_used`, a **window-exact** distinct-app count per
org unit (computed from the app×org weekly slices, so unlike `users_active` it
needs no peak-week approximation; the `-1` unattributed cohort ranks as its own
row). App-scope rank symmetrically carries window-exact
`departments_active_window` / `locations_active_window` columns. The unused
inversion (`include_unused: true`) anti-joins `replica_set` against the window
— cross-check with `usage_last365d = 'No Usage'`; the *difference* between the
two lists (active last year, silent in this window) is itself a finding. For
what to do with an unused-apps result, route to
[[playbooks/application-lifecycle/sunset-funnel]] — window inactivity alone is
never sufficient grounds for deletion (retention rules gate that).

### Variant C — departments/locations active in one app, windowed

`usage_over_window` with `scope: app`, `scope_id`, `by: department` (or
`location`). Manual pattern (weekly app×org cube slices):

```sql
SELECT department_id, max(department_name) AS department,
  sum(user_access_days) AS uad,
  max(users_active) AS users_active_peak_week,
  sum(sessions_total) AS sessions
FROM cqx_data.usage_cube_repset_vs_department_activity_history
WHERE replicaid = '<replicaid>'
  AND week_date >= ('<from_date>'::date - 6) AND week_date <= '<to_date>'::date
GROUP BY department_id
ORDER BY uad DESC
```

Mirror with the location twin. `scope: department|location` + `by: app` flips
the grouping ("top apps used by Sales this year") — subtree-inclusive by
default via `department.path @> ARRAY[<id>]`; when aggregating a subtree, sum
`users_active` across sibling org units *within* each week first, then take the
peak week (disjoint populations within a week; the across-weeks rule is
unchanged).

### Variant D — estate or org-unit trend over a window

`scope: estate` (or `department`/`location` + `scope_id`) in series mode.
Manual: the corresponding weekly table, one row per week, `ORDER BY week_date`.

## Interpretation

- **Sum the additive metrics only** — `user_access_days`, `days_active`,
  sessions, reads, writes are exact across weeks. **Never sum `users_active`**;
  report the peak week and say so. In this profile, exact window-distinct user
  counts are not available at this grain — the aggregate answer and peak-week
  figure are the result; if the customer has already approved the privileged
  MCP posture, exact distinct counts are available via
  `user_activity_over_window`.
- **No tier claims from custom windows** ([[concepts/usage-time-windows]]):
  compare with the embedded canonical figures, never grade a custom total.
- **Whole-week widening** (all weekly variants): iDNA stores usage in whole
  Monday-anchored ISO weeks, so the overlap predicate above widens the window
  to every week it touches — no requested day is lost, but the first and last
  weeks can reach outside the requested dates. The tool reports the real span
  as `covered_from_date` / `covered_to_date`; when running the SQL manually,
  the covered span is the Monday of the from-week through the Sunday of the
  to-week. Quote that span when you report the numbers.
- **History depth is instance-specific** — check `min(week_date)` before
  promising a long window; the tool reports requested vs effective dates.
- **Cross-mode caveats** (variant C): department/location rows are
  **direct-membership grain** (a user belongs to exactly one org unit;
  descendants are separate rows — subtree questions aggregate via `path`);
  **org id `-1` is the unattributed cohort** — activity that could not be
  mapped to an org unit (anonymous access, directory gaps), reported as a row
  rather than dropped. Keep it visible in rankings.

## Variations

- **Strict-mode customer** — everything on this page is aggregate-grain and
  restricted-safe; the attribution framing in
  [[howtos/attribute-usage-aggregate-only]] applies unchanged.
- **Permissive customer, per-user depth wanted** — the per-user counterpart
  howto is visible in permissive sessions only.

## Sources

Backing tables verified against the iDNA ETL (view definitions + additivity
spot-check) 2026-07-27; plan and audit trail in the dev repo.
