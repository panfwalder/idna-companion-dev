---
id: howtos/find-most-used-applications
title: Find the most used applications
type: howto
status: verified
tags: [domain/idna, topic/usage]
related:
  - concepts/user-access-days
  - concepts/focus-db
  - concepts/about-replica-sets
  - concepts/remediation-classification
  - schema/tables/replica-set
  - playbooks/application-lifecycle/lifecycle-routing
  - playbooks/application-lifecycle/keep-funnel
requires_capability: none
source: CoCo Copilot server/knowledge/foundation_domain_knowledge.md
last_reviewed: 2026-04-17
---

# Find the most used applications

## Question

"Which applications are the most used in this environment?"

## Approach

Rank `cqx_data.replica_set` by `user_access_days_last90d` descending, filtered to Focus Replica Sets. "Most used" = highest UAD in the last 90 days — that's the primary metric. Use `NULLS LAST` so applications with no recorded usage don't occupy the top rows.

## Routing note — modernization-prep questions go elsewhere

If "most used / most important" serves a **modernization or migration goal** — a forward target or rollout event such as 64-bit, Nomad, new clients, client rollout/deployment, migration, or upgrade — this UAD ranking is **not** the answer on its own. Route to [[playbooks/application-lifecycle/lifecycle-routing]] → [[playbooks/application-lifecycle/keep-funnel]], which crosses usage with the ETL lifecycle vector (`keep/modernize` vs `sunset`) and per-ruleset findings, and lets you retire sunset candidates to shrink the rollout scope. A pure *"which apps are most used?"* with **no** such target/event cue stays here; a goal phrase alone (*"make sure it works"*) is not enough to switch.

## Data needed

- [[schema/tables/replica-set]] — `replicaid`, `title`, `is_focus_db`, `user_access_days_last90d`, `usage_last90d`, `remediation_classification`, `last_accessed`, `templates_inherits_from`, `design_is_similar_to_standard_template`.

## Procedure

1. State the scope: "Focus Replica Sets, last 90 days, top N." See [[concepts/focus-db]] and [[concepts/user-access-days]].

2. Run the ranking query.

   Intent: return the top N applications by User Access Days over the last 90 days.

   ```sql
   SELECT title, usage_last90d, remediation_classification,
     user_access_days_last90d, last_accessed,
     templates_inherits_from, design_is_similar_to_standard_template
   FROM cqx_data.replica_set
   WHERE is_focus_db
   ORDER BY user_access_days_last90d DESC NULLS LAST
   LIMIT 20
   ```

3. Present with context: state the Focus filter, state the 90-day window, call out applications where `remediation_classification = 'Consolidation Potential'` — those wouldn't normally appear at the top but reading them reassures the filter is sensible.

## Interpretation

- **UAD** combines users × days. A 100-user app used once and a 1-user app used 100 days both score 100. See [[concepts/user-access-days]] for depth.
- **`usage_last90d`** gives a quick usage tier label (Very High → Very Low) for the last 90 days without needing numerical thresholds.
- **NULL `user_access_days_last90d`** — application exists but had no access in the window. Order with `NULLS LAST` to keep them out of the top.

## Variations

- **Longer window** — use `user_access_days_last365d` or `user_access_days_all_time`.
- **Fallback metric** — if UAD is unavailable, rank by `users_active_last90d` on [[schema/tables/usage-summary-by-database]] (per-instance, then aggregate).
- **Per-server breakdown** — drill into [[schema/tables/usage-summary-by-database]] joined to `database`, grouping by `server_name`.
- **By department / location** — swap to [[schema/tables/usage-cube-repset-vs-department]] or its location twin.

## Sources

CoCo Copilot foundation knowledge.
