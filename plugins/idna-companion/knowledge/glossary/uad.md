---
id: glossary/uad
title: UAD
type: glossary
status: verified
tags: [domain/idna, topic/usage]
related:
  - concepts/user-access-days
requires_capability: none
source: CoCo Copilot server/knowledge/foundation_domain_knowledge.md
last_reviewed: 2026-04-17
---

# UAD

**User Access Days.** iDNA's primary usage metric. One unique user opening one application on one unique day = 1 User Access Day. The metric combines user-breadth and time-depth into a single number, so "100 users on 1 day" and "1 user on 100 days" both score 100.

UAD is exposed at the application level on `cqx_data.replica_set` (`user_access_days_last90d`, `_last365d`, `_all_time`) and at the instance level on `cqx_data.usage_summary_by_database`. The 90-day window is the default for "most used" questions.

**See also:** [[concepts/user-access-days]] for depth and fallback metrics.
