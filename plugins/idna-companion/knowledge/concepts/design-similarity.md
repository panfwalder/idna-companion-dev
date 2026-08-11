---
id: concepts/design-similarity
title: Design Similarity
type: concept
status: draft
tags: [domain/idna, topic/modernization, topic/consolidation]
related:
  - concepts/database-complexity
  - concepts/about-template-inheritance
  - schema/topics/similarity
  - schema/topics/template-inheritance
  - schema/tables/replica-set
  - schema/tables/design-insight-overview-bydb
requires_capability: none
source: docs/reference/database-complexity-reviewed.md
last_reviewed: 2026-06-15
---

# Design Similarity

## What it is

iDNA compares the design of analysed databases and groups those that resemble each other. From the similarity scores it builds **clusters** (databases within a configurable similarity threshold — default ≥ 85%; sub-clusters are 100%-identical design), evaluates **template candidates** inside each cluster, and where a candidate is recognised as an HCL **standard template**, flags that relation. This is how iDNA surfaces template lineage even when explicit inheritance has been removed — the design DNA is still detectable.

## Scope of this page

This page owns the **new** similarity material: clusters, sub-clusters, template candidates, and standard-template detection. It does **not** restate the per-database most-similar-template columns or the clean-vs-broken inheritance narrative — those are owned by [[concepts/about-template-inheritance]] and [[schema/topics/template-inheritance]] and only referenced here.

## Why it matters: the complexity corrector

Design similarity is the lens that makes [[concepts/database-complexity]] readable. Because complexity scores the entire design including inherited template bulk, similarity tells you how much of a high score is *inherited* versus *custom*:

- high complexity **+** standard-template relation → largely inherited bulk (a stock mail/discussion/teamroom design plus a little customisation);
- high complexity **without** a standard-template relation → genuinely custom engineering, where redevelopment effort and risk concentrate.

It is also a **consolidation aid**: databases clustering tightly together are candidates for sharing one modernised template or being consolidated, and within a cluster iDNA names the template candidate the others could standardise on.

## Grain rule (load-bearing)

Apply the correction at **replica-set (application) grain** using the columns on `cqx_data.replica_set`: `design_is_similar_to_standard_template`, `most_similar_template_name_*` / `_similarity_*`, `design_cluster_count` / `design_cluster_ids`, `template_similarity_ranking_list`.

The columns `suspected_standard_template_relation`, `similar_standard_templates`, `similarity_pct_standard_templates` live on [[schema/tables/design-insight-overview-bydb]], which is **per-finding, per-DB-instance** grain. Treat them as drill-down evidence to be aggregated/filtered by `replicaid` — never read a single instance's or finding's flag as the application-level verdict.

## Where it appears in the product

Design → Similarity — `{{idna_server}}/#/similarity`: cluster bubbles, sub-clusters (100% identical), and the list of template candidates per cluster, including any recognised standard-template relation.

## Related

- [[concepts/database-complexity]] — the metric this corrects.
- [[concepts/about-template-inheritance]] / [[schema/topics/template-inheritance]] — explicit inheritance and the most-similar-template columns.
- [[schema/topics/similarity]] — the full cross-table field map.

## Sources

`docs/reference/database-complexity-reviewed.md` (reviewed compendium), grounded against the live `cqx_data` schema 2026-06-15.
