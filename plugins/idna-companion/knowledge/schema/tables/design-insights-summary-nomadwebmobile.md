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
  - schema/topics/trial-license-masking
  - schema/topics/insights-overview-exclude
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md; units and trial counts per the ETL trial-findings-surface hand-off (ETL ADR 0011, final contract 2026-10-05); the exclusion per ETL ADR 0012 (2026-10-07)
last_reviewed: 2026-10-07
---

# cqx_data.design_insights_summary_nomadwebmobile

## Purpose

Aggregated finding counts per replica set for the Nomad Web/Mobile compatibility Code Insights ruleset.

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
- For per-finding detail, drill into [[schema/tables/design-insight-overview-byrepset]].

## Notes and caveats

- This table can feed the shared source-code remediation workflow, but the detailed methodological source behind that workflow is still the 64-bit migration docx.
- **Unit: findings, not occurrences.** `count_findings` and the severity columns count distinct rule x code block pairs per app (severity above zero). [[schema/tables/design-insight-overview-byrepset]] rows, and the `exposure_summary` / `most_affected_apps` totals built on them, count occurrences (one per location). On ETL builds with ETL ADR 0012 (2026-10) the summary count equals the byrepset pair count for every app after a settled nightly run ([[schema/topics/insights-overview-exclude]]); on older builds this view still counts apps flagged "Insights Overview Exclude", which the byrepset view leaves out. Never sum or mix the two sources; label the unit.
- **A missing row does not prove "no findings".** An app whose design reference is excluded from the insights analysis ("Insights Overview Exclude") has no row here either, on ETL builds with ETL ADR 0012. Only the flag tells the two apart: [[schema/topics/insights-overview-exclude]].
- **On a trial license** older builds read 0 for every app (the finding surface is unavailable, so a zero is not "no findings"); builds with the trial findings surface carry the full-license numbers for every focus app. See [[schema/topics/trial-license-masking]].

## Sources

CQX schema reference.
