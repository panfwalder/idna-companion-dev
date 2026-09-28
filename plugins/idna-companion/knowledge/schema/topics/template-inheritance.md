---
id: schema/topics/template-inheritance
title: Template Inheritance in the Schema
type: schema
status: verified
tags: [domain/idna, topic/modernization]
related:
  - concepts/about-template-inheritance
  - concepts/database-template
  - concepts/remediation-category
  - concepts/design-similarity
  - schema/topics/similarity
  - schema/tables/replica-set
  - schema/tables/database
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md
last_reviewed: 2026-08-18
---

# Template Inheritance in the Schema

## Purpose

A cross-table guide for working with template inheritance data. The information is split across `replica_set` (application-level, array-valued) and `database` (instance-level, scalar). This page documents the shape differences and common query patterns.

## Where the columns live

### On `cqx_data.replica_set` (application level)

| Column | Type | Meaning |
|---|---|---|
| `templates_inherits_from` | text[] | Explicit template link(s). Non-null indicates a clean association. |
| `templates_inherits_from_count` | integer | Count of explicit parent-template associations. |
| `templates_acts_as_master` | text[] | Template names this replica set acts as master for. |
| `most_similar_template_name_newest_design` | text[] | iDNA-detected similar template when explicit link is missing. |
| `most_similar_template_similarity_newest_design` | numeric | Similarity score for the newest design. |
| `design_is_similar_to_standard_template` | boolean | Quick flag for HCL standard-template derivation. |

Arrays appear here because one replica set can contain instances inheriting from different templates.

### On `cqx_data.database` (instance level)

| Column | Type | Meaning |
|---|---|---|
| `template_inherits_from` | text | Explicit parent template link for this instance. |
| `template_acts_as_master` | text | If set, this instance is a master template. |

Scalars appear here because each instance has at most one explicit parent template.

## Why both exist

- Reporting and planning happen at the application level (`replica_set`).
- Implementation on a specific server happens at the instance level (`database`).
- Master templates are discovered at the instance level; templates are typically not Focus DBs and are not modeled as replica sets for most analysis.

## Clean vs. broken association

- Clean: `replica_set.templates_inherits_from IS NOT NULL`
- Broken: explicit inheritance is missing, but similarity columns point to a likely parent template

That distinction underpins the modernization vector's template routing (`remediate_master`/`upgrade_standard`/`rebase_ancestor` vs. `investigate_inheritance/*` — [[concepts/about-modernization-vector]]; legacy A/B mapping on [[concepts/remediation-category]]).

## Related depth tables

For finer-grained similarity drill-down beyond the summary columns, iDNA provides `cqx_data.similarity_templates` as a depth topic.

## Trial licenses

On trial licenses (2026-08 ETL revision), all template-name columns on this page carry per-run `Master n` aliases for ALL template names (M8 — built-in standard names included); only masters operated solely by trial-selected replica sets stay real. The clean-vs-broken logic and all counts stay honest — only the names are masked. See [[schema/topics/trial-license-masking]].

## Sources

CQX schema reference.
