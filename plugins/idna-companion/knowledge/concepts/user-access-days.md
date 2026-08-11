---
id: concepts/user-access-days
title: User Access Days (UAD)
type: concept
status: verified
tags: [domain/idna, topic/usage]
related:
  - concepts/about-replica-sets
  - concepts/focus-db
  - glossary/uad
  - schema/tables/replica-set
  - schema/tables/usage-summary-by-database
  - schema/tables/usage-cube-repset-vs-user
  - howtos/find-most-used-applications
  - howtos/identify-content-creators-and-consumers
requires_capability: none
source: CoCo Copilot server/knowledge/foundation_domain_knowledge.md
last_reviewed: 2026-04-17
---

# User Access Days (UAD)

## What it is

The primary usage metric in iDNA. One unique user opening one application on one unique day = **1 User Access Day**. UAD combines two usage scenarios into a single comparable number:

- 1 user × 100 days = **100 UAD**
- 100 users × 1 day = **100 UAD**

Both profiles indicate similar access volume; UAD treats them equivalently. A tool with few but devoted users and a tool with broad but thin adoption can both register as high-UAD applications.

## Default evaluation window

**Last 90 days.** When answering "most used" or similar, default to the `_last90d` column unless the user requests otherwise. Always state the time window in the answer.

## Available windows

Per-column variants exist for three time windows:

- `user_access_days_last90d` — rolling 90 days (default)
- `user_access_days_last365d` — rolling 365 days
- `user_access_days_all_time` — since iDNA began collecting

## Where it appears

- **Replica-set level (default):** `cqx_data.replica_set.user_access_days_last90d` etc. See [[schema/tables/replica-set]].
- **Database-instance level (drill-down):** `cqx_data.usage_summary_by_database.user_access_days_last90d` etc. See [[schema/tables/usage-summary-by-database]].
- **Department and location cubes:** `user_access_days` on the cube tables for org-unit analysis.

## Fallback metric

When UAD is not available for a given context, fall back to `users_active_last90d` (distinct user count). State the fallback explicitly.

## Related glossary

- [[glossary/uad]]

## Sources

CoCo Copilot foundation knowledge (Usage Assessment section).
