---
id: schema/tables/design-insights-summary-nomadwebmobile
title: cqx_data.design_insights_summary_nomadwebmobile
type: schema
status: verified
tags: [domain/idna, topic/nomad, topic/modernization]
related:
  - concepts/nomad-web-mobile-compatibility
  - concepts/about-replica-sets
  - schema/topics/cqx-data-execution-surface
  - schema/tables/replica-set
  - schema/tables/design-insight-overview-byrepset
  - playbooks/source-code-remediation/assess-exposure
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md
last_reviewed: 2026-04-17
---

# cqx_data.design_insights_summary_nomadwebmobile

## Purpose

Aggregated finding counts per replica set for the Nomad Web/Mobile compatibility Code Insights ruleset.

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `replicaid` | text | Join key to `replica_set`. |
| `vulnerability_score` | integer | Overall risk score. |
| `count_findings` | integer | Total findings for this application. |
| `findings_verylow` | integer | Severity: Very Low. |
| `findings_low` | integer | Severity: Low. |
| `findings_medium` | integer | Severity: Medium. |
| `findings_high` | integer | Severity: High. |
| `findings_veryhigh` | integer | Severity: Very High. |
| `findings_exceptional` | integer | Severity: Exceptional. |

## Relationships

- `replicaid` -> `cqx_data.replica_set.replicaid`.
- For per-finding detail, drill into [[schema/tables/design-insight-overview-byrepset]].

## Notes and caveats

- This table can feed the shared source-code remediation workflow, but the detailed methodological source behind that workflow is still the 64-bit migration docx.

## Sources

CQX schema reference.
