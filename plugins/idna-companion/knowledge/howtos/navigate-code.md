---
id: howtos/navigate-code
title: Navigate the analysed code - read a design element's code, find who uses it, tell clean code from missing code
type: howto
status: verified
tags: [domain/idna, topic/modernization]
related:
  - schema/tables/code-element-summary
  - schema/tables/code-element
  - schema/tables/code-source
  - concepts/code-block
  - howtos/find-code-patterns
  - howtos/use-source-code-remediation-mcp-tools
  - howtos/find-design-dashboards
  - schema/topics/trial-license-masking
  - schema/topics/insights-overview-exclude
requires_capability: none
source: cqx_data code views and the MCP tools, recipes measured through execute_sql on the panagenda test instance (2026-10-21); search_code contract per docs/plans/active/mcp-code-search (handover 2026-10-21); docs/plans/active/code-navigation
last_reviewed: 2026-10-22
---

# Navigate the analysed code

## Question

"Show me the code of library / agent / form X." "Where is function F defined, and who calls it?" "Who uses library X?" "X has no findings - is it missing from iDNA?" "Show me the rest of the document behind this finding."

## Approach

iDNA stores all analysed code, not only code with findings. Locate code by document or element name in `code_element_summary`, by text in `code_source`, and drill into one database with `code_element`. The finding views and the finding tools answer only "which code matched an insight".

## Data needed

[[schema/tables/code-source]], [[schema/tables/code-element-summary]], [[schema/tables/code-element]]; `cqx_data.replica_set` and `cqx_data.database` (application → instances). On MCP builds with the tool, `search_code` runs these searches for you.

### Name forms

Match a design document or element by its whole key, case-insensitively and literally: `lower(e->>'design_document') = lower('<form>')` (or `code_element`). This finds every spelling of a name (`<FunctionName>` and the same name in other case) and treats `_`, `%`, `"` and `\` as ordinary characters. The only escape is SQL's: double every apostrophe inside the literal, once - the form `agent[@name='O'Neil Tools']` becomes `'agent[@name=''O''Neil Tools'']'`. Use the element's name, not its alias; unnamed documents use `<kind>[n]` (forms and the mapping from a finding row: [[schema/tables/code-element-summary]]). Do not use `ILIKE` or a search in `code_elements::text` for names: wildcards and JSON escaping change the result.

### Scope

Code questions - where code is, who references it, where something is defined - search all analysed databases, focus and non-focus, because a library often sits only in templates or other non-focus databases. Every recipe returns `is_focus_db`; state the scope and mark which results are Focus. Usage, lifecycle and modernization questions keep the Focus default.

## Procedure

1. **Bind the question:** one application (its `replicaid`; it can have several instances, and only focus replica sets have a design reference) or the estate - all analysed databases, focus and non-focus.

2. **Privileged session (or SQL for the customer to run).** With `search_code` (MCP builds with the tool), Q-2 and Q-3 are one call each: `design_document` / `code_element` for Q-2, `pattern` for Q-3; the remaining entries of a row come from its entries request (`database_id` + `code_hash`). Q-1..Q-4 stay for what the tool does not return (an application's full `code_text`, the `named_path` drill-down) and for the customer to run.

   - **Q-1** code of one design document in one application, every instance, the design reference first:

     ```sql
     SELECT
         ces.database_id,
         d.is_focus_db,
         (ces.database_id = rs.database_id_design_reference) AS is_design_reference,
         e->>'design_document' AS design_document,
         e->>'code_element' AS code_element,
         s.hash_id AS code_hash_id,
         s.code_hash,
         s.code_text
     FROM
         cqx_data.replica_set rs
     JOIN
         cqx_data.database d ON d.replicaid = rs.replicaid
     JOIN
         cqx_data.code_element_summary ces ON ces.database_id = d.database_id
     CROSS JOIN LATERAL
         jsonb_array_elements(ces.code_elements) e
     JOIN
         cqx_data.code_source s ON s.hash_id = ces.hash_id
     WHERE
         rs.replicaid = '<replicaid>'
         AND lower(e->>'design_document') = lower('scriptlibrary[@name=''<LibraryName>'']')
     ORDER BY
         is_design_reference DESC NULLS LAST,
         ces.database_id,
         code_element;
     ```

     For one function or event, filter `lower(e->>'code_element') = lower('code[@event=''<FunctionName>'']')` instead of (or in addition to) the document; both filters then apply to the same entry.

   - **Q-2** where a design document sits across the estate (every spelling, literal match):

     ```sql
     SELECT
         d.title,
         d.replicaid,
         ces.database_id,
         d.is_focus_db,
         d.server_name,
         d.file_name,
         count(DISTINCT ces.hash_id) AS code_blocks
     FROM
         cqx_data.code_element_summary ces
     JOIN
         cqx_data.database d ON d.database_id = ces.database_id
     WHERE
         EXISTS (
             SELECT 1
             FROM jsonb_array_elements(ces.code_elements) e
             WHERE lower(e->>'design_document') = lower('scriptlibrary[@name=''<LibraryName>'']')
         )
     GROUP BY
         d.title,
         d.replicaid,
         ces.database_id,
         d.is_focus_db,
         d.server_name,
         d.file_name
     ORDER BY
         d.title;
     ```

     For one function or event: `lower(e->>'code_element') = lower('code[@event=''<FunctionName>'']')` inside the `EXISTS`; for both, both conditions in the same `EXISTS` (the same entry). If it exceeds the tool's time limit, narrow to one application (Q-1).

   - **Q-3** who uses a library or calls a function, across the estate - two calls.

     Step 1, the matching blocks with their stable hash:

     ```sql
     SELECT
         hash_id AS code_hash_id,
         code_hash
     FROM
         cqx_data.code_source
     WHERE
         code_text ILIKE '%use "<LibraryName>"%';
     ```

     Step 2, where they sit - the `code_hash` values from step 1 as a list:

     ```sql
     SELECT
         d.title,
         d.replicaid,
         ces.database_id,
         d.is_focus_db,
         d.server_name,
         d.file_name,
         s.hash_id AS code_hash_id,
         s.code_hash,
         jsonb_array_length(ces.code_elements) AS n_entries,
         jsonb_path_query_array(ces.code_elements, '$[0 to 4]') AS first_entries
     FROM
         cqx_data.code_source s
     JOIN
         cqx_data.code_element_summary ces ON ces.hash_id = s.hash_id
     JOIN
         cqx_data.database d ON d.database_id = ces.database_id
     WHERE
         s.code_hash IN ('<code_hash>', '<code_hash>')
     ORDER BY
         d.title,
         d.server_name,
         d.file_name;
     ```

     One row per database instance and block. Use step 2's `code_hash_id` (the current one) for `get_code_block`. A hash from step 1 with no row in step 2 changed or disappeared in between (a nightly run): run step 1 again. `first_entries` holds up to five summary entries; when `n_entries` is larger, list the rest with the unnest pattern on [[schema/tables/code-element-summary]], filtered by that row's `database_id` and `code_hash_id`. Never select the whole `code_elements` array in an estate-wide search: one row can hold thousands of entries.

     Step 1 returns at most the tool's row limit: when it reports truncation, count first (`SELECT count(*)` with the same `WHERE`) and narrow the pattern before listing. Do not fold step 1 into step 2 as a subquery or a CTE: through `execute_sql` that form is several times slower and can exceed the tool's time limit. For a function: `WHERE code_text ILIKE '%<FunctionName>(%'`; a function without parameters is called without parentheses (`x=<FunctionName>`), so search the bare name as well (with `search_code`, mode `word`) - a zero result for `<FunctionName>(` alone does not mean no caller. Joining `code_source` to `code_element_summary` without the hash list, or searching `code_element` across the estate, is slower still.

   - Q-2 and Q-3 return the instance's title, `replicaid`, `server_name` and `file_name` for opening the database in Domino Designer, and `is_focus_db` for the scope. Several rows with one `replicaid` are replicas of one application.

   - **Designer link** (optional, full licence): add `LEFT JOIN cqx_data.server srv ON srv.server_id = d.server_id` and select `'notesdesigner://' || srv.host_name || '/' || d.replicaid || '/' || (<entry> ->> 'noteid')` for an entry of `first_entries` (or of the unnested entries). On a trial licence `replicaid` is a pseudonym and the built link does not open; use a finding row's `designer_link` where one exists.

   - **Q-4** every code location of one document in one database, with its path inside the document (the drill-down after Q-1 / Q-2; `code_element` must always be filtered by `database_id`):

     ```sql
     SELECT
         ce.named_path,
         ce.language,
         ce.note_data->>'loc' AS lines_of_code,
         ce.code_hash
     FROM
         cqx_data.code_element ce
     WHERE
         ce.database_id = <database_id>
         AND lower(ce.named_path[2]) = lower('form[@name=''<FormName>'']')
     ORDER BY
         ce.named_path;
     ```

   - Read a block found by Q-1..Q-3 with `get_code_block` and its `code_hash_id`, or `code_source.code_text`; from Q-4, `code_source` by `code_hash`.

3. **Restricted session.** With `search_code` (MCP builds with the tool), the same calls as in step 2. On a build without it, this session cannot search code through MCP: point to Custom Insights › Code Search (`{{idna_server}}/#/codesearch`) and the Code Search Results Overview, and offer the SQL above for the customer to run. A `code_hash_id` the user already has (from a finding row or Code Search) still opens with `get_code_block`.

4. **From a finding:** map the finding row's document to the `design_document` form ([[schema/tables/code-element-summary]]); Q-1 then lists the rest of that document's code. Detail rows only: a masked row (trial; `detail_masked = true`, or `database_id_design_reference IS NULL` in the view) carries aliases in place of names - present them as withheld detail, and never join its `code_hash_id` to the code views to recover the names ([[schema/topics/trial-license-masking]]).

## Interpretation

- **An empty finding result is not missing code - and not proof of clean code.** Say "no finding rows were returned by this source and filter". Say "no insight matched" only when all of these hold: the finding source is available for that target (on older trial builds the finding views return nothing although findings exist), the database is in the finding views' scope (a focus replica set's design reference, not flagged "Insights Overview Exclude" ([[schema/topics/insights-overview-exclude]]), inside the parsed set), the ruleset is enabled, and the name is a real name, not a trial alias. Whether the code is stored is a separate question: Q-1 / Q-2 answer it either way. Never say "iDNA does not store it".
- **Element versus mention.** A text search (Q-3) returns every block that mentions the name - the callers' `Use` lines and calls included. The element itself is found by document or element name (Q-1, Q-2).
- **Missing from Q-1 / Q-2:** no analysed code under that name in the databases whose design iDNA could parse. Check the name form (alias instead of name, apostrophe not doubled, `[n]` for unnamed documents) and all instances before concluding. A licensed database outside that set can still list code locations in `code_element`: run Q-4 with its `database_id`. Such a database has no finding analysis, and its blocks may have no `code_source` row; say so instead of reporting the code as absent.
- **Where a function is defined.** The `code[@event='<name>']` key names subs and functions only on newer ETL builds and only where the function has its own code segment; a method inside a class sits in the class's `declarations` segment, and older builds key code by document or `globals[1]`. No matching key is therefore not proof that no definition exists. With `search_code`, a `code_element` key search that finds no row returns the same "no matching key is not proof" note. Then search the text: Q-3 step 1 with `'%function <name>(%'`, `'%sub <name>(%'`, or `'%<name>%'` for a parameterless sub; with `search_code`, `mode: 'regex'` and a pattern such as `(sub|function|property (get|set|let))\s+<name>\y` (PostgreSQL regex: `\y` is the word boundary; `\b` is a backspace, not a boundary). Read the candidate blocks with `get_code_block`, and tell definitions (`Function` / `Sub` / `Property` + the name) from mentions before naming the document.
- **Static references only.** A text search finds literal references: which blocks mention a name, which libraries a block declares with `Use`. Calls assembled at run time (`Evaluate`, `Execute`, agent names in strings, `@Command` arguments) are found only where the literal text appears; same-named subs in different libraries are indistinguishable by text. Zero hits means no literal reference in the analysed code of the licensed databases - not "unused".
- **Search patterns:** `code_text` keeps case but drops comments, indentation and spaces around punctuation and operators: search with `ILIKE` and without those spaces. In `ILIKE`, `%` and `_` are wildcards and `\` escapes: write `\%`, `\_`, `\\` for the literal characters (`search_code` mode `text` takes the pattern literally). Comments cannot be searched.
- **Where, not how.** `named_path` (Q-4) shows where an element sits in the document (table, row, cell, field, event), not how it looks; visual formatting is not in iDNA's code views.
- **Units:** Q-2 counts distinct code blocks per database instance; Q-3 lists blocks per database, each with its number of summary entries (`n_entries`: distinct combinations of design document, code element and note - several raw places can share one entry); Q-4 lists every raw code location, so it can show more places than `n_entries`. `search_code` uses the same entries: for a text search its `n_locations` / `first_locations` are all entries of the block in that database (Q-3's `n_entries` / `first_entries`), for a key search only the entries that match the key; its entries request lists every entry of the (database, block); its Focus field is `is_focus`. Name the unit.
- **Scope:** Q-1..Q-3 cover the analysed code - the licensed databases whose design iDNA could parse, focus and non-focus; `code_element` (Q-4) also covers licensed databases outside that set; finding counts cover focus replica sets. Report the scope and which results are Focus (`is_focus_db`).
- **Large estates:** if an estate-wide search exceeds the tool's time limit, narrow it to one application (Q-1) or a more specific pattern.
- **The original text:** `code_text` is normalized; see [[schema/tables/code-source]].
- **Trial:** the trial licence works per database instance. These views show the licensed instances - selected ones, also in an application that is not fully selected, and non-focus databases licensed by rule - with their real title, server, file and code; unlicensed instances are absent. The application's title in `replica_set` can still read `*** Trial License ***` when not all its instances are selected; that is masking at the application level, not an error. `replicaid` is a pseudonym on trial, the same in every view; built Designer links do not open. Masked finding rows keep their aliases: never recover their names through these views ([[schema/topics/trial-license-masking]]).

## Variations

- Java (`java[@name='<file>']` elements on newer builds; search the class or method name).
- Formula (`'%@DbLookup(%'`).
- Shared elements (`sharedfield`, `sharedactions[n]`).
- "Is this library still referenced anywhere?" - Q-3, with the static-references caveat.
