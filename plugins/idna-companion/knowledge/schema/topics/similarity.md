---
id: schema/topics/similarity
title: Design Similarity in the Schema
type: schema
status: draft
tags: [domain/idna, topic/modernization, topic/consolidation]
related:
  - concepts/design-similarity
  - schema/topics/template-inheritance
  - concepts/about-template-inheritance
  - schema/tables/replica-set
  - schema/tables/design-insight-overview-bydb
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md
last_reviewed: 2026-08-18
---

# Design Similarity in the Schema

## Purpose and ownership boundary

A cross-table field map for design similarity: clustering, template candidates, and standard-template detection. **Ownership boundary:** this page owns the cluster and standard-template-detection columns. The per-database most-similar-template columns (`most_similar_template_name_*`, `_similarity_*`, `design_is_similar_to_standard_template`) and the clean-vs-broken inheritance narrative are owned by [[schema/topics/template-inheritance]] and referenced here, not restated. For what similarity means and how it corrects complexity, see [[concepts/design-similarity]].

## Clustering columns — on `cqx_data.replica_set` (application grain)

| Column | Type | Meaning |
|---|---|---|
| `design_cluster_count` | bigint | Number of design clusters the application participates in. |
| `design_cluster_ids` | integer[] | Identifiers of those clusters. |
| `is_template_candidate` | boolean | This replica set is a candidate template for its cluster. |
| `template_similarity_ranking_list` | text[] | Ranked similar templates/designs. |

The application-level standard-template flag and most-similar-template columns also live here — see [[schema/topics/template-inheritance]] for `design_is_similar_to_standard_template`, `most_similar_template_name_*`, `most_similar_template_similarity_*`.

`is_template_candidate` is also present on `cqx_data.database` at instance grain.

## Standard-template detection — on `cqx_data.design_insight_overview_bydb` (per-finding / per-instance grain)

| Column | Type | Meaning |
|---|---|---|
| `suspected_standard_template_relation` | boolean | iDNA suspects this instance derives from an HCL standard template. |
| `similar_standard_templates` | text[] | Standard template(s) it resembles (set when the flag is true). |
| `similarity_pct_standard_templates` | numeric | Similarity percentage to those templates. |

**Grain rule:** these are drill-down evidence — aggregate by `replicaid` (and `database_id_design_reference` where relevant); do not treat a single finding/instance row as the application verdict. The application-grain signal is on `replica_set`. See [[schema/tables/design-insight-overview-bydb]].

## Depth tables

- `cqx_data.similarity_templates` — finer-grained similarity drill-down beyond the summary columns (also referenced from [[schema/topics/template-inheritance]]). On trial licenses its keys are masked and template names appear as `Master n` aliases (built-in standard names included since M8) (see [[schema/topics/trial-license-masking]]).
- `cqx_data.code_cluster`, `cqx_data.code_similarty` (note the schema's spelling), `cqx_data.code_hash_set` — code-level similarity, the foundation beneath design-level similarity. Use for shared-code-block analysis.

## Typical use

Find the cluster peers / template candidate for an application, then read its standard-template relation at application grain:
```sql
SELECT replicaid, title, design_cluster_ids, is_template_candidate,
       design_is_similar_to_standard_template, most_similar_template_name_newest_design
FROM cqx_data.replica_set
WHERE replicaid = '<replicaid>'
```

## Notes and caveats

- Similarity is a **sibling** signal to complexity, not an input to it: it is how you *correct* complexity for inherited bulk, not part of the complexity calculation.
- Only Focus DBs and templates are analysed, so unanalysed applications have no cluster or similarity data.

## Sources

CQX schema reference; standard-template columns verified populated on the POC instance 2026-06-15.
