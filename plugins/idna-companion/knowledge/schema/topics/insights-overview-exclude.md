---
id: schema/topics/insights-overview-exclude
title: "Insights Overview Exclude: databases left out of the insights analysis"
type: schema
status: verified
tags: [domain/idna, topic/modernization, topic/64bit, topic/nomad]
related:
  - schema/tables/design-insight-overview-byrepset
  - schema/tables/design-insights-summary-notes64bit
  - schema/tables/design-insights-summary-nomadwebmobile
  - schema/tables/database
  - schema/tables/replica-set
  - schema/tables/design-summary-by-replica-set
  - concepts/about-modernization-vector
  - schema/topics/trial-license-masking
requires_capability: none
source: iDNA_Applications_ETL ADR 0012 and the Companion hand-off of its plan insights-overview-exclude (deployed 2026-10-07); ETL dda_dm/scheduled/285 (a kept app without findings routes to `ready`); MCP qualify_app_analysis pattern logic; flag tables per docs/reference/cqx_data_schema_enriched_all_tables.md; the flag query checked live 2026-10-07; implementation audit IX-1..IX-7 (docs/plans/completed/insights-overview-exclude-wiki/)
last_reviewed: 2026-10-07
---

# Insights Overview Exclude: databases left out of the insights analysis

## What it is

"Insights Overview Exclude" is a customer metadata field in iDNA's custom properties. It is set to `Exclude` on a database instance (content type `db`) or on a replica set (content type `rs`); a replica-set flag applies to every database of the replica set. What it removes depends on the ETL build:

- **Every build:** the database is left out of the findings overview and everything built on it.
- **ETL builds with ETL ADR 0012 (2026-10):** it is also left out of the insights summary (score, rank, per-target counts), so it is out of the whole insights analysis.

## What an excluded database looks like

- **Findings overview** (`design_insight_overview_bydb`, [[schema/tables/design-insight-overview-byrepset]]), every build: the database contributes no finding rows. The finding MCP tools return none for it, and the modernization vector, plan and worklist carry no items from it.
- **Insights summary of the instance:** on ETL ADR 0012 builds `database.insights_score` is NULL, `insights_summary` reads `{"insights_score": -1}`, and the Application Insights panel in iDNA is empty. Older builds still compute them.
- **At application level** the insights follow the replica set's design reference (`replica_set.database_id_design_reference`, the database the byrepset rows come from). When the excluded database is that reference:
  - **every build:** the app has no byrepset rows, the finding tools show nothing for it, and a kept focus app routes to `ready` with the display "Ready: no findings for this target" ([[concepts/about-modernization-vector]]);
  - **ETL ADR 0012 builds:** the app also has no row in [[schema/tables/design-insights-summary-notes64bit]] / [[schema/tables/design-insights-summary-nomadwebmobile]]; `design_summary_by_replica_set.insights_score` and `replica_set.design_insights_score` are NULL; the lifecycle tools (`keep_candidates`, `keep_disposition`, `qualify_app_analysis`) report 0 findings, and `qualify_app_analysis` at full depth, with no other diagnostic, then recommends `keep_no_action`;
  - **older builds:** the app keeps its summary row, scores and lifecycle counts, while the findings overview and the finding tools show no findings and the modernization vector reads `ready`. The two sources disagree for that app.

  A flagged instance that is not the design reference changes nothing at application level.
- **Code surfaces stay complete** on every build: `code_source`, `code_element_summary` and the code search's holder counts still include the excluded database's code.

## Excluded or clean? Only the flag tells

An empty finding view or `ready` looks the same for an excluded app and a clean one, and on ETL ADR 0012 builds so do a missing summary row, a NULL score and the `-1` sentinel. Only the flag tells them apart:

```sql
-- Replica sets whose insights an "Insights Overview Exclude" flag removes:
-- a flag on the replica set, or on the database that is its design reference
SELECT rs.replicaid, 'rs' AS flagged_on
FROM cqx_data.custom_replica_set_property p
JOIN cqx_data.custom_property_definition pd
  ON pd.property_id = p.property_id AND pd.content_type = 'rs'
JOIN cqx_data.replica_set rs ON rs.replicaid = p.replicaid
WHERE pd.name = 'Insights Overview Exclude' AND p.value = 'Exclude'
UNION
SELECT rs.replicaid, 'db (design reference)'
FROM cqx_data.custom_db_instance_property p
JOIN cqx_data.custom_property_definition pd
  ON pd.property_id = p.property_id AND pd.content_type = 'db'
JOIN cqx_data.replica_set rs ON rs.database_id_design_reference = p.database_id
WHERE pd.name = 'Insights Overview Exclude' AND p.value = 'Exclude'
```

- **Match the content type.** The ETL ignores a value stored under the other content type's field id.
- **Only the replica set or its design reference counts for the app.** A flag on another instance removes only that instance's own insights, so the query matches `db` flags through `database_id_design_reference`.
- **Trial licenses:**
  - The replica-set key is the published pseudonym, the same `replicaid` the trial views show. Older ETL builds ignored replica-set flags on trial.
  - `database_id_design_reference` is NULL for replica sets outside the trial selection, so the `db` branch finds only flags on trial-selected apps; the `rs` branch covers every app.
- **Access:** the custom-property tables are on the restricted allowlist (ADR 0006 appendix). No MCP tool reports the flag today; where `execute_sql` is available, the query above answers it.

## How to say it

- **Check before calling an app clean.** Before calling one application clean or ready, when the answer depends on its code having been analysed, run the flag check. If the flag is set on the replica set or on its design reference, say so plainly: the application is excluded from iDNA's findings analysis by its "Insights Overview Exclude" setting, so the finding and modernization views show nothing for it. It is the customer's setting; do not present it as an error.
- **Readiness counts:** a readiness count includes excluded kept apps as `ready`. When the flag query shows kept apps, say how many of the ready ones are excluded.
- **Older builds:** summary counts or scores next to an empty finding view can mean an excluded app; the flag explains the difference.

## Consistency of the counts

On ETL ADR 0012 builds the insights summary counts equal the byrepset finding pairs for every app after a settled nightly run, that is, one with no flag edit and no new database during the run. An app whose design reference is excluded is missing from both. A flag edited during a run, or a database new to the catalog in that run, can make the two sources disagree until the next run; the last completed run is the authority. On older builds the summary views still count excluded apps. Either way, never sum or mix the two sources: one counts findings, the other occurrences.
