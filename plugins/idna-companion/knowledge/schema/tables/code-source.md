---
id: schema/tables/code-source
title: cqx_data.code_source
type: schema
status: verified
tags: [domain/idna, topic/modernization]
related:
  - concepts/code-block
  - schema/tables/design-insight-overview-byrepset
  - playbooks/source-code-remediation/classify-shared-code-paths
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md
last_reviewed: 2026-04-17
---

# cqx_data.code_source

## Purpose

The list of unique source-code blocks with their associated hash IDs. Join from finding-level tables via `code_hash_id` to inspect the normalized code behind a finding. See [[concepts/code-block]].

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `hash_id` | integer | Primary key. Referenced elsewhere as `code_hash_id`. |
| `code_hash` | text | Hash string for the normalized block. |
| `code_text` | text | Actual normalized source code (lowercase, comments removed). |
| `note_data` | jsonb | Additional structured metadata for the source block. |

## Relationships

- PK: `hash_id`.
- `hash_id` is joined from `cqx_data.design_insight_overview_byrepset.code_hash_id`.

## Typical use

Retrieve the normalized source for a specific code block:
```sql
SELECT code_text
FROM cqx_data.code_source
WHERE hash_id = <code_hash_id>
```

## Notes and caveats

- `code_text` reflects the hashed payload, not the original Designer formatting.
- For design-element context and navigation links, pair this with [[schema/tables/design-insight-overview-byrepset]].

## Sources

CQX schema reference.
