---
id: schema/tables/design-summary-by-replica-set
title: cqx_data.design_summary_by_replica_set
type: schema
status: verified
tags: [domain/idna, topic/modernization]
related:
  - schema/tables/replica-set
  - schema/tables/replica-set-summary
  - concepts/database-complexity
  - schema/topics/complexity
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md
last_reviewed: 2026-06-15
---

# cqx_data.design_summary_by_replica_set

## Purpose

Per-replica-set design and complexity metrics, including the **per-language code-volume breakdown** that underlies the complexity score. Design analysis anchors to the highest-complexity instance in the set (`database_id_design_reference`). This is the table to use when a question is about *what the design is made of* — how many forms/views/agents, how much LotusScript vs. formula vs. XPages — rather than just the rolled-up grade.

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `replicaid` | text | Application identifier. FK to `cqx_data.replica_set`. |
| `database_id_design_reference` | bigint | Instance whose design is used as the reference for the set. |
| `complexity` | text | Design-complexity label (same family as `replica_set.design_complexity`). |
| `complexity_score` | bigint | Numeric design-complexity score. |
| `insights_score` | bigint | Aggregate code-insights score. |
| `formula_elements` / `formula_loc` | integer / bigint | Formula-language element count and lines of code. |
| `lotusscript_elements` / `lotusscript_loc` | integer / bigint | LotusScript element count and LOC. |
| `javascript_elements` / `javascript_loc` | integer / bigint | JavaScript element count and LOC. |
| `java_elements` / `java_loc` | integer / bigint | Java element count and LOC. |
| `agents_elements` / `agents_loc` | integer / bigint | Agent count and LOC. |
| `forms_elements` / `views_elements` / `folders_elements` / `scriptlibraries_elements` | integer | Per-element-type counts (with matching `_loc`). |
| `xpages_elements` / `xpages_customcontrols` | integer / bigint | XPages elements and custom-control count. |

## Relationships

- Join to `cqx_data.replica_set` on `replicaid`.

## Typical use

Language mix for one application:
```sql
SELECT lotusscript_loc, formula_loc, javascript_loc, java_loc, xpages_customcontrols
FROM cqx_data.design_summary_by_replica_set
WHERE replicaid = '<replicaid>'
```

## Notes and caveats

- **Column-naming divergence.** The per-language volume columns here are **un-prefixed** (`formula_elements`, `xpages_customcontrols`, …). The *same* family on `cqx_data.replica_set` is **`design_`-prefixed** (`design_formula_elements`, `design_xpages_customcontrols`, …). Use the right names for the table you are querying.
- `complexity_score` here and `design_complexity_score` on `replica_set` are the same metric exposed on two tables. For the categorical grade and its ordinal index see [[schema/tables/replica-set-summary]].
- Volumes reflect the **entire** design, including inherited template elements — see the template-inflation caveat in [[concepts/database-complexity]] before reading a high score as custom engineering.

## Sources

CQX schema reference; un-prefixed volume columns verified on the POC instance 2026-06-15.
