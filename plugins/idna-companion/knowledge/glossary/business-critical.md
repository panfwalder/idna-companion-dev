---
id: glossary/business-critical
title: Business Critical
type: glossary
status: verified
tags: [domain/idna, topic/consolidation]
related:
  - concepts/remediation-classification
  - concepts/usage-tier
  - concepts/database-complexity
requires_capability: none
source: derived from concepts/remediation-classification
last_reviewed: 2026-10-23
---

# Business Critical

**A value of `remediation_classification`** on `cqx_data.replica_set`: an application classified as `'Business Critical'` (all-time usage >= High AND complexity >= Medium). The usage side is all-time, so a Business Critical application can have little or no use in the last 90 days. It is precomputed — filter on the classification value rather than reconstructing it from raw usage/complexity columns.

"Most complex" is a separate axis: rank by the numeric `cqx_data.replica_set.design_complexity_score`, distinct from the categorical complexity tier inside this classification.

**See also:** [[concepts/remediation-classification]] for the full rule and the other classification values; [[concepts/usage-tier]] for the usage grades.
