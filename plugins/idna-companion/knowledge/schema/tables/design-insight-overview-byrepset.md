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
  - schema/topics/trial-license-masking
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md; element pair = the code inside the document per ETL ADR 0008 (2026-10-01); Java file names and host events per ETL ADR 0010 (2026-10-02; live on ifa-dev-02); trial rows per the ETL trial-findings-surface hand-off (ETL ADR 0011, final contract 2026-10-05)
last_reviewed: 2026-10-05
---

# cqx_data.design_insight_overview_byrepset

## Purpose

The per-finding detail table for source-code remediation. One row per finding occurrence (rule x code block x location) per replica set. The source for each finding is the database instance with the highest complexity score in that replica set (`database_id_design_reference`). This is where you see which rule matched, in which design element, and which shared code block is involved.

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `replicaid` | text | Join key to `replica_set`. |
| `title` | text | Application title. |
| `title_unique` | text | Disambiguated title. |
| `ruleset_name` | text | Human-readable ruleset name. |
| `finding_name` | text | Which rule matched. |
| `design_document_type` | text | The location's type, one token: `Form`, `View`, `Agent`, `Scriptlibrary`, `Subform`, `Sharedfield`, ... |
| `design_document_name` | text | The design element holding the finding — the location. |
| `code_element_type` | text | What holds the code inside that document: `Code` for code directly on the document, in its Globals, or under a simple-action host (ETL ADR 0008); since ETL ADR 0010 (2026-10-02) `Java` for Java code; otherwise the host element (`Field`, `Action`, `Column`, `Button`, `Outlineentry`, `Pardef`, ...); rarely `Item` for an item such as `$WindowTitle`. Never a location. |
| `code_element_name` | text | For `Code`: an opaque DXL identifier, such as an event (`queryopen`, `selection`, `onLoad`, ...), LotusScript section (`options`, `declarations`), entry sub (`initialize`, `terminate`), sub / function name, or raw `code[1]`. For `Java` (ETL ADR 0010): the file's base name (`JavaAgent.java`, `Utils.java` without the package path) — not unique across documents. Otherwise the host's name; a host without a name carries its event since ETL ADR 0010 (`pardef[4] > hidewhen`, `button[1] > click`, `imageref[1] > value`) — a composed display string, never split it. Not "the function name". |
| `notes_noteid` | text | Note ID for Designer navigation. |
| `designer_link` | text | Deep link to open in Domino Designer. |
| `insight_id` | text | FK to `cqx_data.design_insight`. |
| `ruleset_id` | text | FK to `cqx_data.design_insight_ruleset`. |
| `code_hash_id` | bigint | Shared code-block identifier. |
| `source_code_hash` | text | Hash string representation. |
| `severity` | text | Severity bucket. |
| `severity_weight` | smallint | Numeric severity for ordering. |
| `database_id_design_reference` | bigint | Database instance whose design is used as reference. NULL marks a masked row on trial (see below); on a full license never NULL. |

## Relationships

- `replicaid` -> `cqx_data.replica_set.replicaid`.
- `insight_id` -> `cqx_data.design_insight.insight_id`.
- `ruleset_id` -> `cqx_data.design_insight_ruleset.ruleset_id`.
- `code_hash_id` joins to `cqx_data.code_source.hash_id` for source lookup — a documented join, not an enforced key: on trial a block with no licensed holder (held only by non-selected databases) has no `code_source` row.

## Typical use

Findings for a specific application:
```sql
SELECT finding_name, severity, design_document_type, design_document_name,
  code_element_type, code_element_name, code_hash_id
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
- **One row = one occurrence.** `COUNT(*)` counts occurrences, not distinct findings; a distinct finding is a rule x code block pair within one application and target (`COUNT(DISTINCT (insight_id, code_hash_id))` per app). The insights summary views count findings in that sense — never sum or mix the two sources.
- **On a trial license** the content depends on the build ([[schema/topics/trial-license-masking]]): older builds return no rows; builds with the trial findings surface return detail rows for fully trial-selected replica sets and masked rows for every other focus replica set, so counts cover the whole estate. Row rule: `database_id_design_reference IS NULL` = masked. Masked rows of non-selected apps share the title `*** Trial License ***` (before the first completed nightly run of the trial-findings ETL every app is masked, and the selected apps keep their real title); masked rows carry aliases `<type> <n>` in `design_document_name` / `code_element_name` (scoped to one app, ruleset and type within one ETL run) and NULL `notes_noteid` / `designer_link`. Identify apps by `replicaid` and order by it as the last key, since masked apps share a title; never join or group on the alias names across apps or rulesets.
- `code_hash_id` is the key for shared-code/fix-once analysis (worklist blocks with reach > 1 — see [[schema/tables/modernization-worklist]]; legacy "Category C") and for lookup in [[schema/tables/code-source]].
- **Location = the design document.** The location of a finding is `design_document_type` + `design_document_name`. The code element pair names the code inside that document: `Code` + an opaque DXL identifier (an event, LotusScript section, entry sub, sub / function name, or the raw `code[1]`) for code directly on the document, in its Globals, or under a simple-action host; `Java` + the file's base name for Java code (ETL ADR 0010); otherwise the host element (`Field`, `Action`, `Column`, ...) and its name, with its event appended when the host has no name (`button[1] > click`); rarely `Item` (e.g. `$WindowTitle`). Say "Script Library SetPrevLib, code element CheckAdminAccess" or "Agent (ProcessThumbs), Java file JavaAgent.java"; never "in Code options" as if `Code` were a place, and do not call the name a sub or function unless the code shows it.
- Since 2026-10-01 (ETL ADR 0008) `code_element_type` no longer takes the values `Scriptlibrary`, `Agent`, `Globals`, `Form`, `View`, `Subform`, `Databasescript`, `Folder`, `Page`, `Note`, `Frameset`: filter document kinds on `design_document_type`. An instance whose ETL predates ADR 0008 still shows the document repeated in the element pair (`Scriptlibrary / SetPrevLib / Scriptlibrary / SetPrevLib`) or `Globals / globals[1]`; read the location from the document pair there too and treat it as an older ETL build, not as an error.
- Since 2026-10-02 (ETL ADR 0010): `Java` is a type of its own (file base names are not unique across documents — keep the document pair to name one file); `Code / library` means JavaScript script libraries only and `Code / action` simple actions and formula agents, never Java; bare positional names (`pardef[1]`) no longer occur outside `Code / code[1]`, so an exact filter on one returns nothing — the name is a composed display string (`pardef[1] > hidewhen`), never split or parse it. Identical positional hosts in different places of one document stay indistinguishable. On an ETL build before ADR 0010 Java shows as `Code / library|action` and hosts as bare `pardef[4]`; read the location from the document pair there too.

## Sources

CQX schema reference.
