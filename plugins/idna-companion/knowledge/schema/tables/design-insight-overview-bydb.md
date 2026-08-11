---
id: schema/tables/design-insight-overview-bydb
title: cqx_data.design_insight_overview_bydb
type: schema
status: verified
tags: [domain/idna, topic/modernization, topic/64bit, topic/nomad]
related:
  - schema/tables/design-insight-overview-byrepset
  - schema/tables/replica-set
  - schema/tables/database
  - concepts/design-similarity
  - concepts/about-template-inheritance
  - schema/topics/similarity
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md
last_reviewed: 2026-06-15
---

# cqx_data.design_insight_overview_bydb

## Purpose

The per-finding detail table at **database-instance grain** — one row per finding per DB instance (contrast `design_insight_overview_byrepset`, which collapses to one row per finding per replica set). Besides the finding columns, this is the table that carries iDNA's **standard-template detection** flags, which is why the design-similarity correction depends on it for drill-down evidence.

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `database_id` | bigint | DB instance the finding was observed on. FK to `cqx_data.database`. |
| `replicaid` | text | Application the instance belongs to. FK to `cqx_data.replica_set`. |
| `suspected_standard_template_relation` | boolean | iDNA suspects this instance's design derives from an HCL standard template. |
| `similar_standard_templates` | text[] | The standard template(s) it resembles (populated when the flag is true). |
| `similarity_pct_standard_templates` | numeric | Similarity percentage to the matched standard template(s). |
| `template_reference_direct` | text | Explicit parent-template link for the instance, if any. |
| `finding_name` | text | Which rule matched (finding columns mirror `…_byrepset`). |
| `ruleset_id` | text | FK to `cqx_data.design_insight_ruleset`. Always filter on this. |
| `severity` / `severity_weight` | text / smallint | Severity bucket and numeric weight. |
| `code_hash_id` | bigint | Shared code-block identifier. |

(Finding columns `design_document_type`/`_name`, `code_element_type`/`_name`, `designer_link`, `notes_noteid`, `insight_id`, `source_code_hash` match [[schema/tables/design-insight-overview-byrepset]] — see that page for their use.)

## Relationships

- `replicaid` -> `cqx_data.replica_set.replicaid`.
- `database_id` -> `cqx_data.database.database_id`.
- `insight_id` -> `cqx_data.design_insight.insight_id`; `ruleset_id` -> `cqx_data.design_insight_ruleset.ruleset_id`.

## Typical use

Standard-template relation as drill-down evidence for one application, aggregated up from instance/finding grain:
```sql
SELECT replicaid,
       bool_or(suspected_standard_template_relation) AS any_std_template_relation,
       array_agg(DISTINCT t) FILTER (WHERE t IS NOT NULL) AS templates
FROM cqx_data.design_insight_overview_bydb
LEFT JOIN LATERAL unnest(similar_standard_templates) AS t ON true
WHERE replicaid = '<replicaid>'
GROUP BY replicaid
```

## Notes and caveats

- **Grain.** This table is per-finding, per-DB-instance — it is very large (millions of rows on a real estate) and the standard-template flag repeats across every finding row for an instance. It is **drill-down evidence**, not an application-level verdict.
- **For the application-level template-relation signal, prefer `cqx_data.replica_set`** (`design_is_similar_to_standard_template`, `most_similar_template_*`). Use this table only to inspect *why* / *against which template* at instance grain. See the grain rule in [[concepts/design-similarity]].
- Like `…_byrepset`, this table is not pre-scoped — always filter on `ruleset_id`.

## Sources

CQX schema reference; standard-template columns verified populated on the POC instance 2026-06-15.
