---
id: schema/tables/code-element
title: cqx_data.code_element
type: schema
status: verified
tags: [domain/idna, topic/modernization]
related:
  - schema/tables/code-element-summary
  - schema/tables/code-source
  - howtos/navigate-code
  - howtos/query-modernization-views
  - schema/topics/trial-license-masking
  - howtos/handle-query-timeouts
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md; ETL cqx_data_ifa 010 view definition; columns, path, speed and trial scope verified live 2026-10-19 / 2026-10-21 (docs/plans/active/code-navigation)
last_reviewed: 2026-10-22
---

# cqx_data.code_element

## Purpose

The code locations iDNA holds for the licensed databases, one row each, with the full path inside the design document. It covers more databases than [[schema/tables/code-element-summary]]: also licensed databases whose design iDNA could not parse. Those have no finding analysis, and their blocks may have no [[schema/tables/code-source]] row. Design elements without code are not here.

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `element_id` | bigint | Primary key. |
| `database_id` | bigint | The database instance (→ `cqx_data.database`). **Always filter on it** (see below). |
| `design_document_id` | bigint | The design document (→ `cqx_data.design_element.element_id`). |
| `parent_element_id` | bigint | The host element (→ `cqx_data.design_element.element_id`). |
| `language` | text | `lotusscript`, `formula`, `javascript`, `java`, ... |
| `design_document` | text | The **type** of the design document only (`scriptlibrary`, `form`, `view`) - no name. |
| `code_element` | text | The **type** of the element only (`field`, `action`, `scriptlibrary`) - no name. |
| `note_id` | text | The design note's Notes note ID (hexadecimal). |
| `code_hash` | text | The block (→ `cqx_data.code_source.code_hash`), the stable key. |
| `named_path` | text[] | The location with names - see below. |
| `note_data` | jsonb | `lang` and `loc` (lines of code). |

### `named_path`

- `[1]` the database (`database[@title='...']`), `[2]` the design document in the `code_element_summary` form (`scriptlibrary[@name='<LibraryName>']`), then the host chain - for forms the rich-text containment (`body[1]`, `richtext[1]`, `table[1]`, `tablerow[2]`, `tablecell[1]`, ..., `field[@name='<name>']`) - then `code[@event='<event>']`, then the language segment (`lotusscript[1]`, `formula[1]`, `javascript[1]`).
- It shows where an element sits in the document (table, row, cell, field, event), not how it looks: visual formatting is not in iDNA's code views.
- The path is the raw location on every ETL build.

### The trap

`code_element_summary.code_elements[].design_document` holds the name form; this view's `design_document` column holds only the type. Filter names on `named_path[2]` (the document) or on the path, never on `design_document`.

## Relationships

- `database_id` → `cqx_data.database.database_id`.
- `code_hash` → `cqx_data.code_source.code_hash` (not `hash_id`).
- `design_document_id` / `parent_element_id` → `cqx_data.design_element.element_id`.

## Typical use

**Always filter by `database_id`.** The view is partitioned by database: a `database_id` filter answers in seconds or less, while a filter on `code_hash` or a name alone scans every code location and does not finish within the tool's time limit. For estate-wide questions, find the holders first with `code_element_summary` or `code_source`, then come here per database.

Every location of one block in one database:

```sql
SELECT
    ce.named_path,
    ce.language,
    ce.note_data->>'loc' AS lines_of_code
FROM
    cqx_data.code_element ce
WHERE
    ce.database_id = <database_id>
    AND ce.code_hash = '<code_hash>'
ORDER BY
    ce.named_path;
```

The document drill-down (every code location of one design document) is Q-4 in [[howtos/navigate-code]].

## Notes and caveats

- A raw location count here can exceed the number of summary entries in `code_element_summary`: one entry there can stand for several raw places.
- **Trial.** The licensed instances only, with their real names; the trial licence works per instance ([[schema/topics/trial-license-masking]]).
- `cqx_data.design_element` holds the design elements without code as well; no recipe here.

## Sources

CQX schema reference; ETL view definition; columns, path, speed and the trial scope checked against live data (2026-10-19 / 2026-10-21).
