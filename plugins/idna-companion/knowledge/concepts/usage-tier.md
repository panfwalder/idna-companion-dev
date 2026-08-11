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
requires_capability: none
source: iDNA Usage Category KPIs dashboard (metabase/dashboard/10000010)
last_reviewed: 2026-06-15
---

# Usage Tier

## What it is

iDNA assigns each replica set a **usage tier** — a qualitative grade of how heavily the application is being used over a given time window. Tiers are computed from access-day counts (see [[concepts/user-access-days]]) bucketed into six labels.

The bucket cut-offs are **not fixed numbers**: iDNA derives them from UAD **percentiles within each customer's own environment**, so "High" for a 50k-user estate is a very different access-day count than "High" for a 100-user estate. There is therefore no static threshold table — the per-category reference values for a given environment are shown on the **Usage Category KPIs** dashboard (`{{idna_server}}/metabase/dashboard/10000010`).

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

## Continuity note: `usage_category_simplified`

An older `usage_category_simplified` column may still appear in the schema for backward compatibility with existing queries and dashboards. New work should rely on `usage_last90d` instead.

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

iDNA usage-tier computation (UAD percentiles, computed per environment). The **Usage Category KPIs** dashboard (`{{idna_server}}/metabase/dashboard/10000010`) is the reference surface for the actual per-category values; cut-offs are environment-relative, so no static threshold table exists.
