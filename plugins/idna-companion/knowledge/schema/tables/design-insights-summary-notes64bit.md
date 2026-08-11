---
id: schema/tables/design-insights-summary-notes64bit
title: cqx_data.design_insights_summary_notes64bit
type: schema
status: verified
tags: [domain/idna, topic/64bit, topic/modernization]
related:
  - concepts/notes-64bit-compatibility
  - concepts/about-replica-sets
  - schema/topics/cqx-data-execution-surface
  - schema/tables/replica-set
  - schema/tables/design-insight-overview-byrepset
  - playbooks/source-code-remediation/assess-exposure
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md
last_reviewed: 2026-04-17
---

# cqx_data.design_insights_summary_notes64bit

## Purpose

Aggregated finding counts per replica set for the Notes 64-bit compatibility Code Insights ruleset. Use this for exposure sizing and for ranking the most-affected applications before drilling into detail.

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
- For individual finding detail, drill into [[schema/tables/design-insight-overview-byrepset]].

## Typical use

```sql
SELECT COUNT(DISTINCT dis.replicaid) AS affected_applications,
  SUM(dis.count_findings) AS total_findings,
  SUM(dis.findings_veryhigh) AS very_high,
  SUM(dis.findings_high) AS high
FROM cqx_data.design_insights_summary_notes64bit dis
JOIN cqx_data.replica_set rs USING (replicaid)
WHERE rs.is_focus_db
```

## Notes and caveats

- This is a summary table. For rule-level detail (`finding_name`, `code_element_type`, `code_hash_id`), use the overview-byrepset table.
- There is no `ruleset_id` column here; the table choice implies the ruleset.

## Sources

CQX schema reference.
