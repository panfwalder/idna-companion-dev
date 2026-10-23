---
id: schema/tables/code-source
title: cqx_data.code_source
type: schema
status: verified
tags: [domain/idna, topic/modernization]
related:
  - concepts/code-block
  - schema/tables/code-element-summary
  - schema/tables/code-element
  - howtos/navigate-code
  - schema/tables/design-insight-overview-byrepset
  - playbooks/source-code-remediation/classify-shared-code-paths
  - schema/topics/trial-license-masking
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md; trial semantics per ETL ADR 0009 and the ETL trial-findings-surface hand-off (ETL ADR 0011, final contract 2026-10-05); code storage, normalization and the original text verified live 2026-10-19 (docs/plans/active/code-navigation)
last_reviewed: 2026-10-22
---

# cqx_data.code_source

## Purpose

Every unique code block of the analysed code - the licensed databases whose design iDNA could parse - with or without findings: the finding views hold only code that matched an insight. [[schema/tables/code-element-summary]] shows where a block sits; a finding row's `code_hash_id` leads to the code behind a finding. Licensed databases whose design iDNA could not parse are not covered here; [[schema/tables/code-element]] can still list their code locations. See [[concepts/code-block]].

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `hash_id` | integer | Primary key. Referenced elsewhere as `code_hash_id`. Run-local (see below). |
| `code_hash` | text | Hash string for the normalized block; the stable key. |
| `code_text` | text | Normalized source: comments and indentation removed, spaces around punctuation and operators removed (`x = Lbound ( a )` is stored as `x=Lbound(a)`), case kept. |
| `note_data` | jsonb | Holder ids and language of the block (no source text). |

## Relationships

- PK: `hash_id`.
- `hash_id` is joined from `cqx_data.design_insight_overview_byrepset.code_hash_id` — a documented join, not an enforced key (see the trial caveat below).
- `cqx_data.code_element_summary.hash_id` → `hash_id`: where the block sits.
- `cqx_data.code_element.code_hash` → `code_hash`: every code location with its path.
- **Keys:** `hash_id` joins `code_source`, `code_element_summary` and the finding views within one ETL run; `code_hash` is the stable key across runs and the key of `code_element` and `code_block_hcl_origin`.

## Typical use

Retrieve the normalized source for a specific code block:
```sql
SELECT code_text
FROM cqx_data.code_source
WHERE hash_id = <code_hash_id>
```

Search the code text case-insensitively (`ILIKE`) and write the pattern without spaces around punctuation or operators: `'%use "<LibraryName>"%'`, `'%<FunctionName>(%'`. In `ILIKE`, `%` and `_` are wildcards and `\` escapes: write `\%`, `\_`, `\\` for the literal characters. Comments cannot be searched. For a search across the estate, search `code_source` first and look up the hits afterwards - [[howtos/navigate-code]] has the recipes.

## Notes and caveats

- **An empty finding result is not missing code.** A block without a finding row is still here; to check whether code exists, look it up here and in `code_element_summary`, never in the finding views.
- **The original text.** `code_text` is the normalized form. iDNA may also keep the original text, with comments and formatting, compressed in `cqx_data.code_source_raw`; whether it is there varies by block and language, and the recipes on these pages cannot read it as text. For the original as written, use the detail view of Custom Insights › Code Search or open the design element in Domino Designer.
- For the finding context of a block (rule, severity, navigation links), pair this with [[schema/tables/design-insight-overview-byrepset]].
- **Ids are run-local.** `hash_id` (`code_hash_id` elsewhere) is renumbered by every ETL run, so a numeric id can be reassigned to another block; `code_hash` (`source_code_hash` in the finding views) is the stable key. Retain the hash, and obtain the current id from a current finding or modernization row before a lookup.
- **On a trial license, source is withheld when a block has no licensed holder** (ETL ADR 0009): this table keeps only blocks held by at least one licensed database. A finding row can therefore reference a `code_hash_id` with no row here. For an id verified from a masked finding row in the current completed ETL run, a missing row (`get_code_block` NOT_FOUND) means "code withheld on trial", not an invalid finding id; otherwise a missing row alone does not distinguish withheld source from an unresolved id. The code of licensed instances is visible here also when their application is not fully selected for the trial. See [[schema/topics/trial-license-masking]].

## Sources

CQX schema reference; code storage, normalization and the original text checked against live data (2026-10-19).
