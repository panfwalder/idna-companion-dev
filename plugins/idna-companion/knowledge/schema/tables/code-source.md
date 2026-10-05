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
  - schema/topics/trial-license-masking
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md; trial semantics per ETL ADR 0009 and the ETL trial-findings-surface hand-off (ETL ADR 0011, final contract 2026-10-05)
last_reviewed: 2026-10-05
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
- `hash_id` is joined from `cqx_data.design_insight_overview_byrepset.code_hash_id` — a documented join, not an enforced key (see the trial caveat below).

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
- **Ids are run-local.** `hash_id` (`code_hash_id` elsewhere) is renumbered by every ETL run, so a numeric id can be reassigned to another block; `code_hash` (`source_code_hash` in the finding views) is the stable key. Retain the hash, and obtain the current id from a current finding or modernization row before a lookup.
- **On a trial license, source is withheld when a block has no licensed holder** (ETL ADR 0009): this table keeps only blocks held by at least one licensed database. A finding row can therefore reference a `code_hash_id` with no row here. For an id verified from a masked finding row in the current completed ETL run, a missing row (`get_code_block` NOT_FOUND) means "code withheld on trial", not an invalid finding id; otherwise a missing row alone does not distinguish withheld source from an unresolved id. See [[schema/topics/trial-license-masking]].

## Sources

CQX schema reference.
