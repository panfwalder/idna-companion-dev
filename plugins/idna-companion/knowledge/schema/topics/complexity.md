---
id: schema/topics/complexity
title: Complexity in the Schema
type: schema
status: draft
tags: [domain/idna, topic/modernization]
related:
  - concepts/database-complexity
  - schema/tables/replica-set
  - schema/tables/replica-set-summary
  - schema/tables/design-summary-by-replica-set
  - schema/tables/database
  - schema/tables/dbviews-used
  - schema/topics/similarity
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md
last_reviewed: 2026-06-22
---

# Complexity in the Schema

## Purpose

A cross-table guide to where iDNA's design-complexity data lives and which column to use for which question. For what complexity *means* and how to read it, see [[concepts/database-complexity]]; this page is the field map.

## Computed complexity columns

| Table | Columns | Notes |
|---|---|---|
| `cqx_data.replica_set` | `design_complexity`, `design_complexity_simplified`, `design_complexity_score`, `design_complexity_display_index` | Application grade + numeric score. The wide application table. |
| `cqx_data.database` | `design_complexity_score` | **Source grain** — per-instance numeric score, where complexity originates. See [[schema/tables/database]]. |
| `cqx_data.replica_set_summary` | `complexity_category`, `complexity_category_simplified`, `complexity_index` | Compact label + ordinal sort key. See [[schema/tables/replica-set-summary]]. |
| `cqx_data.design_summary_by_replica_set` | `complexity`, `complexity_score` | Label + score alongside the per-language volumes. See [[schema/tables/design-summary-by-replica-set]]. |

## Grain and provenance

Complexity is computed **per database instance** (`cqx_data.database.design_complexity_score`). The application-level value on `cqx_data.replica_set` is **inherited from the reference instance** — `database_id_design_reference`, the highest-complexity instance in the set — **not** summed or averaged across replicas (verified: the replica-set score equals the reference instance's score). The `replica_set`, `replica_set_summary`, and `design_summary_by_replica_set` values are therefore the same metric surfaced at application grain; only the database instance is the origin. The categorical grade (Very High … Not Analyzed) is assigned at the replica-set / summary level — the instance carries only the raw numeric score.

## Grade labels and ordinal

Observed labels (verified 2026-06-15): **Very High / High / Medium / Low / Very Low / Not Analyzed**. Simplified family: **Very High / Medium / High → "Medium / High" / Low / Very Low → "Low / Very Low" / Not Analyzed**. `complexity_index` maps the simplified family to an ordinal: `0 = Not Analyzed`, `1 = Low / Very Low`, `2 = Medium / High`, `3 = Very High`. When ranking by complexity, exclude `Not Analyzed` / index `0`.

## Per-language code volumes — naming divergence

The element/LOC breakdown appears on two tables with **different naming**:

| Table | Naming | Examples |
|---|---|---|
| `cqx_data.replica_set` | `design_`-prefixed | `design_formula_elements`, `design_lotusscript_loc`, `design_xpages_customcontrols` |
| `cqx_data.design_summary_by_replica_set` | un-prefixed | `formula_elements`, `lotusscript_loc`, `xpages_customcontrols` |

Languages/element types covered: formula, lotusscript, javascript, java, agents, folders, forms, views, scriptlibraries, xpages (elements + LOC; XPages also has `customcontrols`). Use the correct names for the table you query.

## Manual field — not the computed score

`complexity_estimation` (on `cqx_data.replica_set` and `cqx_data.database`) is **not** the computed complexity. It sits in the customer-annotation block (`effort_estimation`, `cost_estimation`, `cost_center`, `business_owner`, `technical_owner`) and is sparsely populated — a manually-entered planning field (inference). Do not read it as `design_complexity_score`.

## Element-usage evidence hooks (limited)

- `cqx_data.environment_kpis.views_defined` / `views_indexed` / `views_storage` — an environment-wide rollup. Per-app/per-instance view-index usage is now available via [[schema/tables/dbviews-used]] (used = built index), with aggregate counts in `cqx_data.database.database_views` (aggregate across a replica set's instances for an application-level read).
- `cqx_data.database_content_analysis.form_details` — per-Form document counts from content analysis; heavy and separately licensed per database, so rarely populated. See the usage-evidence tiers in [[concepts/database-complexity]].

## Notes and caveats

- Complexity reflects the whole design, including inherited template bulk; correct for it via the similarity columns mapped in [[schema/topics/similarity]].
- Only Focus DBs and templates are analysed, so most rows are `Not Analyzed`.

## Sources

CQX schema reference; labels, ordinal mapping, and column naming verified on the POC instance 2026-06-15.
