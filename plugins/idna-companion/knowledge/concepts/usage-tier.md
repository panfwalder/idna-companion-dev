---
id: concepts/usage-tier
title: Usage Tier
type: concept
status: verified
tags: [domain/idna, topic/usage]
related:
  - concepts/user-access-days
  - concepts/about-replica-sets
  - concepts/database-complexity
  - schema/tables/replica-set
  - concepts/lifecycle-vector
  - concepts/usage-time-windows
  - howtos/find-usage-dashboards
  - concepts/remediation-classification
requires_capability: none
source: iDNA Usage Category KPIs dashboard (metabase/dashboard/10000010); ETL dda_dm/scheduled/230 (percentile assignment) and 280 (usage_category_simplified)
last_reviewed: 2026-10-23
---

# Usage Tier

## What it is

iDNA assigns each replica set a **usage tier** — a qualitative grade of how heavily the application is being used over a given time window. Tiers are computed from access-day counts (see [[concepts/user-access-days]]) bucketed into six labels.

The bucket cut-offs are **not fixed numbers**: iDNA derives them from UAD **percentiles within each customer's own environment**, so "High" for a 50k-user estate is a very different access-day count than "High" for a 100-user estate. There is therefore no static threshold table — the per-category reference values for a given environment are shown on the **Usage Category KPIs** dashboard (Usage › Usage Category KPIs, `{{idna_server}}/#/dashboard/10000010`).

**How a set gets its label.** For each horizon iDNA takes five reference values: the 97th, 90th, 80th, 65th and 40th percentile of UAD over the focus replica sets with any use in all recorded history. Each replica set gets the label of the reference value its UAD is nearest to (Very High, High, Medium, Low, Very Low in that order; a tie goes to the lower label); 0 UAD is No Usage. If all five values are 0, every set with use in that horizon is Very Low. On a thinly used estate most of those focus sets have 0 UAD in a short horizon, so the lower reference values are all 0 and the labels collapse. Example: with the 97th percentile at 50 UAD and the others at 0, every set above 25 UAD is Very High and every set from 1 to 25 is Very Low; High, Medium and Low do not occur. A jump such as 27 UAD Very High next to 22 UAD Very Low is this rule, not an error.

## The six tier labels

Ordered from heaviest to no use:

- Very High
- High
- Medium
- Low
- Very Low
- No Usage

When a playbook page writes *"≥ Medium"* or *"below Low"* without further qualification, it operates on these labels.

## The family of horizon columns

iDNA computes the tier across multiple time windows. Each window has its own tier column on `cqx_data.replica_set`:

| Horizon | Column |
|---|---|
| Last 7 days | `usage_last7d` |
| Last 30 days | `usage_last30d` |
| Last 90 days | `usage_last90d` |
| Last 365 days | `usage_last365d` |
| All recorded history | `usage_all_time` |

Each column carries one of the six tier labels for its respective horizon.

## Canonical column for unqualified "usage tier"

When a playbook or recommendation refers to *"the usage tier"* or *"Medium usage"* without specifying a horizon, **the canonical column is `usage_last90d`**. The 90-day window is the operational default for lifecycle decisions.

This convention was set when the tier computation shifted from the older simplified field to the 90-day-window-based label.

## `usage_category_simplified` - the classification's usage input

`usage_category_simplified` is a three-band simplification of `usage_all_time`: `High / Very High`, `Low / Medium` (also when the set had any use in the last 90 days) and `No / Low`. It is not a tier for usage questions - use `usage_last90d` for those - but it is the usage input of [[concepts/remediation-classification]], so read it (or `usage_all_time`) when explaining why an application is, for example, Business Critical with little recent use.

## Common patterns

- **Fast lifecycle partition by recent activity.** `usage_last90d >= 'Medium'` is the keep/modernize cut used by [[playbooks/application-lifecycle/keep-funnel]].
- **Disagreement across horizons is itself a diagnostic.** An app with `usage_last90d = 'No Usage'` but `usage_all_time = 'High'` is either declining toward sunset or seasonally periodic. The lifecycle qualify analysis treats this disagreement as the entry point to per-app diagnosis.

## Related

- [[concepts/user-access-days]] — the underlying access-day metric.
- [[schema/tables/replica-set]] — the columns themselves and their full schema.
- [[concepts/usage-time-windows]] — what to do when the question names a window
  that is *not* one of these horizons ("last six months"): custom windows exist
  (weekly grain, both profiles), they enrich rather than replace the canonical
  buckets, and tiers must never be assigned from them.

## Sources

iDNA usage-tier computation (UAD percentiles, computed per environment): ETL `dda_dm/scheduled/230_dda_dm.dbusage_summary_by_replica_set.update.sql` (the percentile reference values and the nearest-value assignment) and `280_dda_dm.ai_replicaset_catalog.update.sql` (`usage_category_simplified`); the collapse observed on the panagenda test instance, 2026-10-23. The **Usage Category KPIs** dashboard (Usage › Usage Category KPIs, `{{idna_server}}/#/dashboard/10000010`) is the reference surface for the actual per-category values; cut-offs are environment-relative, so no static threshold table exists.
