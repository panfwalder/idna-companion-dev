---
id: schema/tables/modernization-vector
title: cqx_data.modernization_vector
type: schema
status: verified
tags: [domain/idna, topic/modernization, topic/64bit, topic/nomad]
related:
  - concepts/about-modernization-vector
  - concepts/design-families
  - concepts/lifecycle-vector
  - schema/tables/modernization-plan
  - schema/tables/modernization-worklist
  - schema/tables/replica-set
  - schema/tables/design-insight-overview-byrepset
  - schema/topics/trial-license-masking
  - howtos/query-modernization-views
requires_capability: none
source: ETL cqx_data_ifa/010_cqx_data.initialize_components_structure.pxsql (view) over dda_dm.modernization_vector (ETL script 285); types verified against live instance 2026-07-23; dismissed columns verified live 2026-07-24; trial masking + detail_masked gated live on dev instances 2026-08-17/18
last_reviewed: 2026-08-18
---

# cqx_data.modernization_vector

## Purpose

Per-app modernization readiness: the routing verdict ([[concepts/about-modernization-vector]]), its human-readable display, session assignment, coverage/shared-solo profile, design-family context, and the finding items — one row per **(kept focus application × included compatibility ruleset)**.

## GRAIN WARNING (the #1 query error on this view)

**One row per app per ruleset — with the two built-in targets, every app appears TWICE.** Unfiltered aggregation double-counts: 574 rows = 287 apps × 2 rulesets on a reference instance. This is *not* the instance-vs-replica-set distinction (the view is already at replica-set grain) and *not* a data defect — readiness is per-target by design. Every aggregate needs a `ruleset_id` predicate (or `GROUP BY ruleset_id`, or a knowing `count(DISTINCT replicaid)`). Mixing targets in one breakdown is semantically wrong, not just doubled.

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `replicaid` | text | Join key to `replica_set`. |
| `title`, `title_unique` | text | Application title (disambiguated). |
| `lifecycle_vector` | text | The app's lifecycle value (always `keep/%` here — see cohort note). |
| `ruleset_id`, `ruleset_name` | text | The compatibility target this row scores. **Filter on it.** |
| `modernization_vector` | text | The raw routing value (see the value tree in [[concepts/about-modernization-vector]]). |
| `modernization_vector_display` | text | User-facing sentence — quote it, don't paraphrase. |
| `modernization_search` | text | Vector padded to full depth with trailing slash, for prefix filtering (see below). |
| `session_kind`, `session_anchor` | text | Which plan session clears this app (join to `modernization_plan`). |
| `n_fix_items` | integer | Distinct code BLOCKS to fix (not items entries — see `items` note). |
| `n_covered_by_master` | integer | Blocks a master refresh clears. NULL when master unscanned. |
| `n_diverging` | integer | Blocks where the app diverges from the master (refresh would overwrite — review first). NULL when master unscanned. |
| `n_surplus` | integer | Blocks beyond the master's design (residual work). NULL when master unscanned. |
| `n_shared_items` | integer | Blocks whose hash also occurs on other focus apps (estate-wide reach > 1). |
| `n_solo_items` | integer | Blocks unique to this app estate-wide (reach = 1). |
| `master_names` | text[] | Master template(s) the app inherits from. |
| `master_scanned` | boolean | Primary master's design collected? `false` → coverage columns NULL, qualifier `/unscanned_master_template`. |
| `most_similar_template`, `most_similar_pct` | text, numeric | Best similarity target for investigate routes. |
| `design_family_id` | integer | Family key (see [[concepts/design-families]]). |
| `design_family` | text[] | ALL family member replicaids — **including sunset apps**. |
| `design_family_peers`, `peers_keep_modernize`, `peers_keep_qualify`, `peers_sunset` | integer | Peer counts by lifecycle. |
| `user_access_days_last90d`, `user_access_days_last365d` | integer | Usage context. |
| `items` | jsonb | One entry per ACTIVE finding (see size warning below). |
| `n_dismissed_items` | integer | Count of config-dismissed non-issue occurrences (`= jsonb_array_length(dismissed_items)`; occurrence grain — deliberately different from the block-grain `n_fix_items`). |
| `dismissed_items` | jsonb | Findings dismissed as non-issue via instance configuration — documented, counted nowhere (see below). |
| `detail_masked` | boolean | TRUE = trial license AND repset not trial-selected: item identity fields are masked (see trial section). FALSE on full license. Since the 2026-08 ETL revision. |

## `modernization_search` (prefix filtering)

The vector padded to full depth with a **trailing slash** (`template/remediate_master/fully_covered_by_master/`, `template/unify_similar/design_family_#8/`). Built for "starts with": prefix `template/remediate_master/` matches the bare value plus all qualifier variants; the trailing slash prevents `design_family_#8` from matching `#80`. This is the intended way to select "everything under node X". The raw `modernization_vector` is NOT padded — a bare `template/remediate_master` there means "fully covered".

## `items` shape and size warning

Array with **one entry per ACTIVE finding** — a code block can appear with several insights. Fields per entry: `type`, `name` (design document), `element_type`, `element_name`, `code_hash_id`, `code_hash`, `insight_id`, `finding_name`, `noteid`, `designer_link`, `clearance` (`covered` | `diverging` | `surplus`).

- `n_fix_items` counts **distinct code blocks**; unpacking `items` without `DISTINCT code_hash` inflates block counts.
- **`items` can be very large** (mail-template apps carry hundreds of findings). Query defensively: project specific fields, use `jsonb_array_length(items)` before unpacking, add `LIMIT`. Do not SELECT whole `items` columns across many rows.

```sql
-- Distinct blocks for one app, defensively
SELECT DISTINCT item ->> 'code_hash' AS code_hash, item ->> 'clearance' AS clearance
FROM cqx_data.modernization_vector mv, jsonb_array_elements(mv.items) AS item
WHERE mv.replicaid = '<replicaid>' AND mv.ruleset_id = '<ruleset_id>'
```

## Dismissed non-issues (`dismissed_items` / `n_dismissed_items`)

Admins can declare finding-grain false positives in the instance config
(`etl_modernization_ignore_codeblocks`, CSV of `code_hash|insight_id` pairs; `|*` = every insight on that block). Dismissed findings count in **NO computation** — routing/`ready`, all counts, reach, plan KPIs, worklist rows — but since the 2026-07 revision they are **documented, not invisible**: each suppressed occurrence appears in `dismissed_items` with the same fields as an `items` entry, minus `clearance`, plus `assessment: 'non_issue'`.

- **Reconciliation:** raw finding occurrences = `items` entries + `dismissed_items` entries, per (app × ruleset). `n_dismissed_items` is occurrence-grain — do NOT compare it with the block-grain `n_fix_items`.
- An app whose findings are ALL dismissed is `ready`, and its display says so ("… finding(s) dismissed as non-issue (instance configuration)").
- The config carries no who/when/why — dismissals are instance configuration, not attributed decisions.
- Scope: kept cohort apps only (a dismissal matching only non-cohort apps surfaces nowhere).
- **Availability:** the two columns ship with the 2026-07 ETL revision; instances on older builds do not have them.
- **MCP surfacing:** from the 2026-08 MCP build, `modernization_for_app` reports `n_dismissed_items` in its core `vector` payload — no SQL needed for the count; query `dismissed_items` (below) only when the individual dismissals matter.

## Cohort / NULL semantics

- Rows exist for **`keep/%` focus apps only**. Sunset apps have **no rows** — but they DO count in shared-block reach and appear in `design_family` arrays and `peers_sunset`. An app absent from this view is out of scope (sunset or non-focus), **not** implicitly ready.
- Coverage columns (`n_covered_by_master`/`n_diverging`/`n_surplus`) are **NULL when the master is unscanned** — unknown, not zero. The consultant action is *scan the master* (place it on a Domino server iDNA scans), not *celebrate zero divergence*.

## Typical use

Readiness split for one target:
```sql
SELECT modernization_vector, count(*) AS apps
FROM cqx_data.modernization_vector
WHERE ruleset_id = '<ruleset_id>'
GROUP BY 1 ORDER BY 2 DESC
```

Everything under one category (prefix filter):
```sql
SELECT title, modernization_vector_display
FROM cqx_data.modernization_vector
WHERE ruleset_id = '<ruleset_id>' AND modernization_search LIKE 'template/remediate_master/%'
```

## Trial licenses (2026-08 ETL revision)

On `ia_eval_license` instances, computation stays honest (routing, counts, KPIs are real) but identity detail is masked — full rules in [[schema/topics/trial-license-masking]]:

- `items`/`dismissed_items`: `name`/`element_name` are ordinal aliases (`Form 2`) and `noteid`/`designer_link` null unless the repset is trial-selected; `code_hash*`, `insight_id`, `finding_name`, `clearance` always populated. `detail_masked` marks such rows — present them as "detail withheld, a full license unlocks it", NEVER as "no work".
- ALL template names in `session_anchor`, `master_names`, `most_similar_template`, and the display are per-run `Master n` aliases — built-in standard names included (M8); only masters operated solely by trial-selected replica sets stay real. The `(standard template)`/`(standard master)` suffixes still mark the class. Aliases renumber each run but are join-consistent across vector/plan/worklist within a run.
- Older trial builds instead show a false all-clear (all rows `ready`, 0 items) — suspect the build, not the estate.

## Notes and caveats

- Built-in ruleset ids: Notes 64bit `71f16a9a-8fc6-4dfe-add8-3a43c539d353`, Nomad `a8efac35-3e54-48f4-aa27-4c342529ed0a`; custom rulesets can be included via configuration.
- `n_shared_items`/`n_solo_items` measure **estate-wide exact-hash reach across ALL focus apps including sunset** — NOT "shared within the design family". Family siblings are similar by design, not by hash; drifted script versions have different hashes (a family member can be 8/0 while its sibling is 4/4).
- For the two built-in targets, `cqx_data.replica_set` carries headline columns (`modernization_vector_notes64bit[_display]`, `modernization_vector_nomad[_display]`) for quick per-app lookups.
