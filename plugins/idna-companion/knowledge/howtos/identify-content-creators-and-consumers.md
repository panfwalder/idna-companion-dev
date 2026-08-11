---
id: howtos/identify-content-creators-and-consumers
title: Identify content creators and consumers (per-user, permissive)
type: howto
status: verified
tags: [domain/idna, topic/usage, topic/regulatory, audience/consultant]
related:
  - playbooks/application-lifecycle/keep-funnel
  - playbooks/application-lifecycle/stakeholder-cohort
  - howtos/analyze-usage-by-department
  - howtos/attribute-usage-aggregate-only
  - concepts/regulatory-usage-constraints
  - concepts/user-access-days
  - concepts/about-replica-sets
  - schema/tables/replica-set
  - schema/tables/usage-cube-repset-vs-user
  - schema/tables/repset-activity-summary-per-user
  - schema/tables/usage-sessions
requires_capability: mcp_unrestricted
source:
last_reviewed: 2026-05-08
---

# Identify content creators and consumers (per-user, permissive)

## Question

"For one application's heavy-user cohort, who is reading versus writing — which users are content creators (material writes) and which are consumers (high reads, low writes)?" Per-user attribution at the application grain. Available at customers where per-user analysis is permitted; suppressed at strict customers.

This is the **permissive-mode** counterpart to [[howtos/attribute-usage-aggregate-only]]. The framing for which mode applies is in [[concepts/regulatory-usage-constraints]]. For the upstream cohort selection (which application(s) deserve this drill-down), see [[playbooks/application-lifecycle/stakeholder-cohort]] and [[playbooks/application-lifecycle/keep-funnel]]. For application-grain single-most-recent reader/writer (restricted-safe, no cohort), see [[schema/tables/replica-set]].

## Approach

Use the per-user pre-aggregated cube `cqx_data.usage_cube_repset_vs_user` keyed on `(replicaid, username)`. The cube exposes both read and write counters (`reads_client`, `writes_client`) so a single query yields the read/write split. Drop into time-period or session-level detail only when the cube does not expose the grain you need.

## Data needed

- `cqx_data.usage_cube_repset_vs_user` — per-user pre-aggregated cube; primary source.
- `cqx_data.repset_activity_summary_per_user` — time-period split (30d / 90d / all-time).
- `cqx_data.usage_sessions` — raw session detail keyed on `person_id`.

## Procedure

1. Confirm the customer's regulatory mode is permissive per [[concepts/regulatory-usage-constraints]]. If strict, refuse and route to [[howtos/attribute-usage-aggregate-only]].

2. Query `cqx_data.usage_cube_repset_vs_user` filtered by `replicaid`, ordered by `user_access_days` descending, limited to a working set (e.g. top 25):

```sql
SELECT u.username,
  u.user_access_days,
  u.sessions_total,
  u.reads_client,
  u.writes_client,
  u.last_access,
  u.last_write_access,
  u.department_name,
  u.location_name
FROM cqx_data.usage_cube_repset_vs_user u
WHERE u.replicaid = '<replicaid>'
ORDER BY u.user_access_days DESC NULLS LAST
LIMIT 25
```

3. Classify each row by the read/write split:
   - **Creator** — material writes (`writes_client` non-trivial relative to peers); the user contributes documents.
   - **Consumer** — high reads, low/zero writes; the user depends on the application without contributing.
   - **Mixed** — both axes non-trivial; commonly the application owners and most-engaged stakeholders.

## Variations

- **Time-period split (30d / 90d / all-time per user).** Switch to `cqx_data.repset_activity_summary_per_user` (`username`, `last_access`, `last_write_access`, `sessions_total`, `sessions_write`, `time_period`).
- **Raw session-level detail.** Drop into `cqx_data.usage_sessions` (keyed on `person_id`).

## Interpretation

- A heavy-legacy keeper dominated by **consumers** is a candidate for archive-with-retention (the data is still being read; the application no longer receives new content).
- A keeper dominated by **creators** is a current keeper or a modernize candidate, depending on the design-side findings; route to [[playbooks/source-code-remediation/remediation-routing]] if findings are material.
- A keeper with no creators in the last 365d but high all-time UAD is a consumption-only application; combine with [[playbooks/application-lifecycle/sunset-funnel]] to test whether retention has it locked.

## Sources

User's blog post on stakeholder framing (creators vs consumers, Read / Write distinction). Schema foundation: [[schema/tables/replica-set]], [[schema/tables/usage-cube-repset-vs-department]] (peer-cube context).
