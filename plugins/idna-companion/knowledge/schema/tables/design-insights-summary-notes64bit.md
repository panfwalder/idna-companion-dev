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
  - schema/topics/trial-license-masking
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md; element pair = the code inside the document per ETL ADR 0008 (2026-10-01); units and trial counts per the ETL trial-findings-surface hand-off (ETL ADR 0011, final contract 2026-10-05)
last_reviewed: 2026-10-05
---

# cqx_data.design_insights_summary_notes64bit

## Purpose

Aggregated finding counts per replica set for the Notes 64-bit compatibility Code Insights ruleset. Use this for exposure sizing and for ranking the most-affected applications before drilling into detail.

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `replicaid` | text | Join key to `replica_set`. |
| `vulnerability_score` | integer | Overall risk score: a severity-weighted analysis value, not a count. |
| `count_findings` | integer | Findings for this application: distinct rule x code block pairs, not occurrences. |
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

- This is a summary table. For rule-level detail (`finding_name`, `design_document_type` / `_name`, `code_hash_id`), use the overview-byrepset table.
- There is no `ruleset_id` column here; the table choice implies the ruleset.
- **Unit: findings, not occurrences.** `count_findings` and the severity columns count distinct rule x code block pairs per app (severity above zero). [[schema/tables/design-insight-overview-byrepset]] rows, and the `exposure_summary` / `most_affected_apps` totals built on them, count occurrences (one per location). On current data the summary count matches the byrepset pair count per app (measured, not guaranteed), except for apps flagged "Insights Overview Exclude", which the byrepset view honours and this view does not. Never sum or mix the two sources; label the unit.
- **On a trial license** older builds read 0 for every app (the finding surface is unavailable, so a zero is not "no findings"); builds with the trial findings surface carry the full-license numbers for every focus app. See [[schema/topics/trial-license-masking]].

## Sources

CQX schema reference.
