---
id: schema/tables/replica-set
title: cqx_data.replica_set
type: schema
status: verified
tags: [domain/idna, topic/usage, topic/consolidation, topic/modernization]
related:
  - concepts/about-replica-sets
  - concepts/replica-id
  - concepts/focus-db
  - concepts/user-access-days
  - concepts/usage-tier
  - concepts/about-template-inheritance
  - concepts/remediation-classification
  - concepts/lifecycle-vector
  - concepts/about-modernization-vector
  - schema/tables/modernization-vector
  - concepts/database-complexity
  - concepts/design-similarity
  - schema/tables/replica-set-summary
  - schema/tables/design-summary-by-replica-set
  - schema/topics/complexity
  - schema/topics/similarity
  - schema/topics/cqx-data-execution-surface
  - schema/tables/usage-summary-by-database
  - schema/tables/design-insights-summary-notes64bit
  - schema/tables/design-insights-summary-nomadwebmobile
  - schema/tables/usage-cube-repset-vs-department
  - schema/tables/usage-cube-repset-vs-location
  - schema/tables/usage-cube-repset-vs-user
  - schema/topics/trial-license-masking
  - schema/tables/repset-activity-summary-per-user
  - howtos/identify-content-creators-and-consumers
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md; the declared-link gloss per code-block-eval-fixes Part E / PE-4 (option b) and mcp-trial-awareness EX-11, 2026-10-05
last_reviewed: 2026-10-05
---

# cqx_data.replica_set

## Purpose

One row per replica set, i.e. one row per distinct application in the environment. This is the primary application-level table for usage, template inheritance, remediation classification, design-complexity rollups, and Focus-DB scoping. Join other application-level tables here on `replicaid`.

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `replicaid` | text | Replica ID, the stable application identifier. |
| `title` | text | Human-readable application title. |
| `db_type_id` | integer | FK to `cqx_data.database_inventory`. |
| `db_type_name` | text | Human-readable database-type label. |
| `is_focus_db` | boolean | iDNA Focus flag. Default scope filter. See [[concepts/focus-db]]. |
| `replicas_count` | integer | Number of database instances in the replica set. |
| `server_names` | text[] | Servers hosting replicas. |
| `usage_category_simplified` | text | Pre-calculated usage tier (Very High to Very Low). |
| `usage_last7d` | text | Pre-calculated usage tier (Very High to Very Low) in the last 7 days. |
| `usage_last30d` | text | Pre-calculated usage tier (Very High to Very Low) in the last 30 days. |
| `usage_last90d` | text | Pre-calculated usage tier (Very High to Very Low) in the last 90 days. |
| `usage_all_time` | text | Pre-calculated usage tier (Very High to Very Low) across recorded history. |
| `user_access_days_last7d` | integer | UAD in the last 7 days. |
| `user_access_days_last30d` | integer | UAD in the last 30 days. |
| `user_access_days_last90d` | integer | UAD in the last 90 days. Primary "most used" metric. |
| `user_access_days_last365d` | integer | UAD in the last 365 days. |
| `user_access_days_all_time` | integer | UAD across recorded history. |
| `users_active_last90d` | integer | Distinct active users in the last 90 days. |
| `sessions_last90d` | bigint | Session count in the last 90 days. |
| `last_accessed` | date | Most recent read/open. |
| `last_accessed_user` | text | Last user recorded as reader. |
| `last_accessed_on_server` | text | Server of the most recent read. |
| `last_write_access` | date | Most recent write. |
| `last_write_user` | text | Last user recorded as writer. |
| `remediation_classification` | text | iDNA application triage. Key value `"Consolidation Potential"`. |
| `lifecycle_vector` | text | ETL-computed lifecycle bucket, `<direction>/<disposition>[/<reason>]`. NULL for non-focus. The lifecycle MCP tools read this. See [[concepts/lifecycle-vector]]. |
| `modernization_vector_notes64bit` | text | Modernization readiness headline for the Notes 64-bit target. NULL unless a kept focus app. See [[concepts/about-modernization-vector]]; full detail on [[schema/tables/modernization-vector]]. |
| `modernization_vector_notes64bit_display` | text | User-facing sentence for the 64-bit vector — quote it, don't paraphrase. |
| `modernization_vector_nomad` | text | Modernization readiness headline for the Nomad Web/Mobile target. NULL unless a kept focus app. |
| `modernization_vector_nomad_display` | text | User-facing sentence for the Nomad vector. |
| `templates_inherits_from_count` | integer | Count of explicit parent-template associations. |
| `templates_inherits_from` | text[] | Declared template link(s). A clean association only when an estate-operated master carries the name. |
| `templates_acts_as_master` | text[] | Template names this replica set acts as master for. |
| `most_similar_template_name_newest_design` | text[] | iDNA-detected similar template when explicit link is missing. |
| `most_similar_template_similarity_newest_design` | numeric | Similarity score for the newest design. |
| `design_is_similar_to_standard_template` | boolean | Quick flag for HCL standard-template derivation. |
| `design_complexity_score` | bigint | Aggregate design-complexity score. Read it through template similarity — see [[concepts/database-complexity]]. |
| `design_insights_score` | bigint | Aggregate code-insights score. |
| `database_id_design_reference` | bigint | Database instance used as design reference. |
| `is_content_age_analyzed` | boolean | Whether content-age analysis has completed for this replica set. Retention and doc-age fields below are populated only when true. |
| `contentage_status` | text | Status of the content-age scan. |
| `retention_analysis_status_detail` | text | Detail/diagnostic string for the retention-analysis outcome. |
| `retention_config_status` | text | Retention-config consistency across the set's instances: `ok` / `missing` / `conflict`. |
| `retention_config` | text | Effective retention configuration, from iDNA Custom DB Properties. |
| `retention_cutoff` | date | Retention horizon cutoff date. |
| `docs_in_retention_max` | bigint | Max documents inside the retention window across the set's instances. |
| `docs_in_retention_min` | bigint | Min documents inside the retention window across instances. |
| `docs_total_max` | bigint | Max total document count across instances. |
| `docs_total_min` | bigint | Min total document count across instances. |
| `has_docs_in_retention` | boolean | Whether any instance holds documents inside the retention window. |
| `retention_analysis_status_lifecycle_annotation` | text | Annotation feeding the lifecycle_vector reason segment (e.g. warning / scan-failure reasons). |
| `is_selected_trial_db` | boolean | TRUE = the replica set is trial-selected: its modernization item detail AND its identity (real `title`, licensed `replica_paths`, pseudo-bracket `title_unique` — M9) stay unmasked on trial licenses. FALSE on full license. Appended last; since the 2026-08 ETL revision — older builds lack it. See [[schema/topics/trial-license-masking]]. |

## Relationships

- PK: `replicaid`.
- `db_type_id` -> `cqx_data.database_inventory.db_type_id`.
- `replicaid` joins to `cqx_data.design_insights_summary_notes64bit`, `cqx_data.design_insights_summary_nomadwebmobile`, `cqx_data.design_insight_overview_byrepset`, `cqx_data.usage_cube_repset_vs_department`, and `cqx_data.usage_cube_repset_vs_location`.
- To aggregate from instance-level views, join `database` on `replicaid`.
- Replica-set template columns are arrays because multiple database instances in the set may inherit from different templates.

## Typical use

- "How many applications does this environment have?" -> `COUNT(*) WHERE is_focus_db`
- "Most used applications" -> order by `user_access_days_last90d DESC NULLS LAST`
- "Consolidation candidates" -> filter on `remediation_classification = 'Consolidation Potential'`
- Scoping join for any finding or usage query that starts from an application

## Notes and caveats

- Usage metrics are exposed consistently across 7/30/90/365-day and all-time windows, so match the question's horizon instead of hard-coding 90 days when the user asks for something else.
- `templates_inherits_from` is an array; use array operators (`= ANY`, `ARRAY_LENGTH`) rather than plain equality.
- `user_access_days_*` columns may be NULL for applications with no recorded usage; use `NULLS LAST` when ordering.
- This table is pre-scoped to applications. Master templates live on `cqx_data.database`, not here. See [[concepts/database-template]].
- Architectural context: this table sits on the public `cqx_data` query surface rather than an internal ETL staging schema. See [[schema/topics/cqx-data-execution-surface]].
- The retention / content-age fields (`retention_*`, `docs_*`, `has_docs_in_retention`, `contentage_status`) are populated only when content-age analysis has run (`is_content_age_analyzed = true`), otherwise NULL. The `min` / `max` pairs are the replica-set roll-up across its instances; a `min != max` divergence is what feeds the lifecycle `conflict_retention` reason (see [[concepts/lifecycle-vector]]).
- The `modernization_vector_*` headline columns cover the two **built-in** targets only; custom compatibility rulesets surface exclusively through `cqx_data.modernization_vector`. Headline values exist only for kept (`keep/%`) focus apps — NULL is "out of modernization scope", not "ready".
- `last_accessed_user` and `last_write_user` are application-grain point-in-time columns: they name the **single** most-recent reader and writer respectively, not a ranking. For a heavy-user cohort or a read/write split across multiple users, use the per-user cube `cqx_data.usage_cube_repset_vs_user` (permissive only — see [[howtos/identify-content-creators-and-consumers]]). These columns are restricted-safe: pseudonymized when the customer's license sets `pseudonymize=Y`; real values when the customer has waived pseudonymization. See [[concepts/regulatory-usage-constraints]] for the strict / permissive framing.
- TRIAL licenses (2026-08 ETL revision): the template-name columns (`templates_inherits_from`, `templates_acts_as_master`, `most_similar_template_name_*`, `template_similarity_ranking_list`) carry per-run `Master n` aliases for ALL template names except masters operated solely by trial-selected replica sets (M8 — built-in standard names are aliased too; the old `*** Not Licensed ***` sentinel arrays are gone, and the `*_count` twins are honest distinct-name counts matching the arrays). Titles/paths/replicaids are pseudonymized as before. See [[schema/topics/trial-license-masking]].

## Sources

CQX schema reference.
