---
id: schema/tables/code-element-summary
title: cqx_data.code_element_summary
type: schema
status: verified
tags: [domain/idna, topic/modernization]
related:
  - schema/tables/code-source
  - schema/tables/code-element
  - concepts/code-block
  - howtos/navigate-code
  - schema/tables/design-insight-overview-byrepset
  - schema/tables/database
  - schema/topics/trial-license-masking
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md; ETL dda_dwh 061 and dda_dm 083; structure, name forms, scope and trial filter verified live 2026-10-19 / 2026-10-21 (docs/plans/active/code-navigation)
last_reviewed: 2026-10-22
---

# cqx_data.code_element_summary

## Purpose

Where each code block sits: one row per database instance and block, listing the block's summary entries in that database (see Reading `code_elements` below). Covers the analysed code - the licensed databases whose design iDNA could parse, focus and non-focus - with or without findings; the findings come from the same set.

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `database_id` | bigint | The database instance (→ `cqx_data.database`). |
| `hash_id` | integer | The block (→ `cqx_data.code_source.hash_id`). Run-local, like `code_source.hash_id`. |
| `code_elements` | jsonb | Array of entries; each entry has `noteid`, `design_document` and `code_element`. |

### Name forms

- **`design_document`** is the design note's DXL element name in lower case, then either a name or a position:
  - named documents: `<kind>[@name='<name>']`, for example `scriptlibrary[@name='<LibraryName>']` - the name as written (case kept, inner apostrophes as they are, no alias);
  - unnamed documents: `<kind>[n]` - every shared-actions document (`sharedactions[1]`), the database script (`databasescript[1]`), the help documents (`helpaboutdocument[1]`, `helpusingdocument[1]`), `note[<n>]`, and the rare unnamed form or view (`form[1]`).
  - Kinds seen on the panagenda test estates (observed, not exhaustive): `form`, `view`, `folder`, `subform`, `sharedactions`, `scriptlibrary`, `agent`, `outline`, `page`, `sharedfield`, `frameset`, `databasescript`, `helpaboutdocument`, `helpusingdocument`, `sharedcolumn`, `note`, `appletresource`.
  - This form is the same on every ETL build.
- **`code_element`** is the code inside the document, an opaque display string as on the finding rows: `code[@event='<name>']` (an event, a LotusScript section, sub or function), `java[@name='<file>']`, or a host with its event such as `pardef[4] > hidewhen`; hosts use their own attribute (`action[@title='<title>']`, `field[@name='<name>']`). These forms exist since ETL ADR 0008 / 0010 (2026-10); on older builds the entry repeats the document, or shows `globals[1]`, `code[@event='library']` or a bare positional host. Filter on `design_document` where you can: it is the same on every build.

### Mapping from a finding row

From a row of [[schema/tables/design-insight-overview-byrepset]]:

- **Document:** when `design_document_name` has the shape `<kind>[n]` (`sharedactions[1]`, `databasescript[3]`), use it verbatim; otherwise lower-case `design_document_type` and wrap the name: `Scriptlibrary` + `<LibraryName>` gives `scriptlibrary[@name='<LibraryName>']`.
- **Element** (ETL builds with ADR 0008 / 0010): `Code` + `<name>` gives `code[@event='<name>']`, `Java` + `<file>` gives `java[@name='<file>']`, `Field` + `<name>` gives `field[@name='<name>']`, `Action` + `<title>` gives `action[@title='<title>']`; a positional host name (`pardef[1] > hidewhen`, `button[2] > click`) is used verbatim.
- Double every apostrophe inside the SQL literal.
- Detail rows only: a masked trial row carries an alias (`Form 7`), not a name - never map or join it to recover a name ([[schema/topics/trial-license-masking]]).

### Reading `code_elements`

- One row is one code block in one database. Each entry is a distinct combination of design document, code element and note where the block sits. Several entries are the same code in several places (the same formula in two views, for example).
- One entry can also stand for several raw places that share that combination - the same code in a document and in its Globals, or repeated unnamed hosts in one document - so the number of entries is not a count of raw locations. Every raw location, with its path, is in [[schema/tables/code-element]] (filter by `database_id`).
- `noteid` is the design note's Notes note ID in hexadecimal, the value finding rows carry as `notes_noteid`. Entries with the same `noteid` are places in the same design note.
- A `[n]` form is a position, not a name: `note[3]` is the third design note of that kind in this database's design export, `code[1]` the first code segment without an event name. Do not compare positions across databases; describe them as unnamed ("an unnamed design note, note[3]"), never as a document called "3".
- `note[n]` holds design notes that the design export does not name by type; on the test estates they hold Java code, for example generated web-service classes.
- One entry can stand for many blocks: one design note can hold hundreds of Java files, each its own block under the same `code[1]` entry. The block is `hash_id`; never merge blocks because their entries match.
- A common block (a short formula) can have thousands of entries in one database. Select `jsonb_array_length(code_elements) AS n_entries` and a slice (`jsonb_path_query_array(code_elements, '$[0 to 4]') AS first_entries`) in searches across many rows; the full array belongs in a query for one `database_id` and `hash_id`.
- Match an entry by its whole key, case-insensitively and literally: `lower(e->>'design_document') = lower('<form>')`. This finds every spelling of a name and treats `_`, `%`, `"` and `\` as ordinary characters. Never match names with `ILIKE` or in `code_elements::text`.
- The MCP tool `search_code` (MCP builds with the tool) uses the same entries: for a text search its `n_locations` / `first_locations` are all entries of the block in that database, for a key search only the entries that match the key; its entries request lists every entry of the (database, block).

### Not the same as `code_element`

The `design_document` key here holds the name form; the `design_document` column of [[schema/tables/code-element]] holds only the type.

## Relationships

- `database_id` → `cqx_data.database.database_id` (instance title, server, file, `replicaid`, `is_focus_db`).
- `hash_id` → `cqx_data.code_source.hash_id` (the code; the stable `code_hash`).
- One row per (`database_id`, `hash_id`).

## Typical use

The recipes live in [[howtos/navigate-code]]. The unnest pattern for one database:

```sql
SELECT
    ces.database_id,
    e->>'design_document' AS design_document,
    e->>'code_element' AS code_element,
    ces.hash_id AS code_hash_id
FROM
    cqx_data.code_element_summary ces
CROSS JOIN LATERAL
    jsonb_array_elements(ces.code_elements) e
WHERE
    ces.database_id = <database_id>
ORDER BY
    design_document,
    code_element;
```

For all entries of one block in one database, add `AND ces.hash_id = <code_hash_id>`.

## Notes and caveats

- **Scope.** Focus and non-focus databases of the parsed set; the finding views hold focus replica sets only. Databases flagged "Insights Overview Exclude" keep their rows here ([[schema/topics/insights-overview-exclude]]).
- **Ids are run-local** (as in [[schema/tables/code-source]]); `code_hash` is the stable key.
- **Replicas.** An instance's code can differ from its replicas: list the application's instances and prefer the design reference where one exists (`replica_set.database_id_design_reference`, focus replica sets only).
- **Same-named documents.** Two documents with the same name in one database share one name form.
- **Trial.** The licensed instances only, with their real names; the trial licence works per instance ([[schema/topics/trial-license-masking]]).

## Sources

CQX schema reference; ETL view and table definitions; structure, name forms, scope and the trial filter checked against live data (2026-10-19 / 2026-10-21).
