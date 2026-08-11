---
id: schema/tables/design-insight-overview-byrepset
title: cqx_data.design_insight_overview_byrepset
type: schema
status: verified
tags: [domain/idna, topic/modernization, topic/64bit, topic/nomad]
related:
  - concepts/code-block
  - concepts/remediation-category
  - concepts/design-similarity
  - schema/tables/design-insight-overview-bydb
  - concepts/notes-64bit-compatibility
  - concepts/nomad-web-mobile-compatibility
  - schema/tables/replica-set
  - schema/tables/code-source
  - schema/tables/design-insights-summary-notes64bit
  - schema/tables/design-insights-summary-nomadwebmobile
  - playbooks/source-code-remediation/classify-template-paths
  - playbooks/source-code-remediation/classify-shared-code-paths
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md
last_reviewed: 2026-07-24
---

# cqx_data.design_insight_overview_byrepset

## Purpose

The per-finding detail table for source-code remediation. One row per finding per replica set. The source for each finding is the database instance with the highest complexity score in that replica set (`database_id_design_reference`). This is where you see which rule matched, in which design element, and which shared code block is involved.

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `replicaid` | text | Join key to `replica_set`. |
| `title` | text | Application title. |
| `title_unique` | text | Disambiguated title. |
| `ruleset_name` | text | Human-readable ruleset name. |
| `finding_name` | text | Which rule matched. |
| `design_document_type` | text | Form, View, Agent, Script Library, etc. |
| `design_document_name` | text | Design element containing the finding. |
| `code_element_type` | text | Subroutine/function/event-handler type. |
| `code_element_name` | text | Name of the code element. |
| `notes_noteid` | text | Note ID for Designer navigation. |
| `designer_link` | text | Deep link to open in Domino Designer. |
| `insight_id` | text | FK to `cqx_data.design_insight`. |
| `ruleset_id` | text | FK to `cqx_data.design_insight_ruleset`. |
| `code_hash_id` | bigint | Shared code-block identifier. |
| `source_code_hash` | text | Hash string representation. |
| `severity` | text | Severity bucket. |
| `severity_weight` | smallint | Numeric severity for ordering. |
| `database_id_design_reference` | bigint | Database instance whose design is used as reference. |

## Relationships

- `replicaid` -> `cqx_data.replica_set.replicaid`.
- `insight_id` -> `cqx_data.design_insight.insight_id`.
- `ruleset_id` -> `cqx_data.design_insight_ruleset.ruleset_id`.
- `code_hash_id` joins to `cqx_data.code_source.hash_id` for source lookup.

## Typical use

Findings for a specific application:
```sql
SELECT finding_name, severity, code_element_type, design_document_name, code_hash_id
FROM cqx_data.design_insight_overview_byrepset
WHERE replicaid = '<replicaid>' AND ruleset_id = '<ruleset_id>'
ORDER BY severity_weight DESC
```

High-impact shared code blocks:
```sql
SELECT code_hash_id, COUNT(DISTINCT replicaid) AS affected_apps
FROM cqx_data.design_insight_overview_byrepset
WHERE ruleset_id = '<ruleset_id>'
GROUP BY code_hash_id
HAVING COUNT(DISTINCT replicaid) > 1
ORDER BY affected_apps DESC
```

## Notes and caveats

- Always filter on `ruleset_id`; this table is not pre-scoped the way the summary tables are.
- `code_hash_id` is the key for shared-code/fix-once analysis (worklist blocks with reach > 1 — see [[schema/tables/modernization-worklist]]; legacy "Category C") and for lookup in [[schema/tables/code-source]].

## Sources

CQX schema reference.
