---
id: schema/topics/trial-license-masking
title: Trial-license masking across cqx_data
type: schema
status: verified
tags: [domain/idna, topic/modernization]
related:
  - schema/tables/modernization-vector
  - schema/tables/modernization-plan
  - schema/tables/modernization-worklist
  - schema/tables/template-ancestry
  - schema/tables/modernization-lineage-dossier
  - schema/tables/modernization-session-steps
  - schema/tables/replica-set
  - schema/topics/template-inheritance
  - schema/topics/similarity
  - concepts/about-modernization-vector
requires_capability: none
source: ETL cqx_data_ifa/010_cqx_data.initialize_components_structure.pxsql + dda_dm scripts 280/285/286/288; behavior gated live on dev instances 2026-08-17/18 (ETL repo, plans modernization-trial-license + modernization-trial-refinement); template-lineage views per ETL 010 view definitions and ADRs 0002-0004 (2026-09-24); lineage dossier document grain per ETL ADR 0005 (2026-09-25); element pair = the code inside the document per ETL ADR 0008 (2026-10-01)
last_reviewed: 2026-10-02
---

# Trial-license masking across cqx_data

## What trial mode is

iDNA instances on an **evaluation/trial license** (`ia_eval_license` instance property; an ABSENT key also means trial) pseudonymize customer-identifying data across `cqx_data`. Full-license instances are entirely unaffected by everything on this page except two appended columns (see Availability).

## Identity pseudonymization

- `replicaid` values are deterministic 16-hex pseudonyms (`md5` of the raw id) — stable across runs, so joins between views keep working. The pseudonym is NEVER unmasked, not even for trial-selected replica sets.
- Titles read `*** Trial License ***`; `title_unique` reads `*** Trial license (<pseudonym>) ***`; replica paths and mail-in names are masked — EXCEPT fully trial-selected replica sets (M9 revision): they show their real `title`, their LICENSED instances' real `replica_paths`, and a real `title_unique` with the pseudonym in the bracket (`AutoMover [<pseudonym>]`). `mailin_names` and the newest/oldest-design columns stay masked even for selected repsets.

## Honest compute + detail masking (2026-08 ETL revision)

Modernization outputs are **computed on real data** — routing values, session structure, all counts, reach, and KPIs are honest. What is masked is *identity detail*:

- In `modernization_vector.items` / `dismissed_items`, the fields `name` / `element_name` become per-app ordinal aliases (the type plus a per-run ordinal: `name` e.g. `Form 2` or `Scriptlibrary 1`, `element_name` e.g. `Code 3` or `Field 1`) and `noteid` / `designer_link` are null — UNLESS the app's replica set is **trial-selected**. `code_hash_id`, `code_hash`, `insight_id`, `finding_name`, `clearance` stay populated everywhere.
- `modernization_vector.detail_masked` (boolean) marks exactly those rows. **Treat `detail_masked = true` as "detail withheld — a full license unlocks it", never as "no work".**
- `replica_set.is_selected_trial_db` (boolean) marks the trial-selected replica sets (the handful of apps chosen at trial setup; their item detail AND — since M9 — their identity: real title/paths/unique title stay unmasked).
- Worklist `item_display` / `sample_location` show real document names only when the source replica set is trial-selected; otherwise the same ordinal-alias form.

## Master-name aliasing (2026-08 ETL revision)

ALL template names (acts-as-master, inherits-from references, AND built-in standard names — M8 revision) appear as **per-run ordinal aliases `Master n`**. The ONLY exception: masters operated solely by trial-selected replica sets keep their real name. Neither built-in-standard recognition nor customer-configured `custom_known_databases` patterns clear a name; the standard/estate CLASS stays visible through the display labels and routing values instead. Surfaces:

- `modernization_vector`: `modernization_vector_display`, `session_anchor`, `master_names[]`, `most_similar_template`, `ancestor_name`
- `modernization_plan`: `anchor`, `item_display` ("Remediate master Master 292…", "Rebase master Master 17 onto Master 4 lineage (98% similar)")
- `modernization_worklist` (+ `_by_session`): `sessions`, `session_anchors[]`, `authored_in`, `master_targets[]`, `session_anchor`
- `template_ancestry`: `tmpl_name`, `ancestor_name`, `latest_name` — shipped standard names included; `family_name` and versions stay clear
- `modernization_lineage_dossier`: `tmpl_name`
- `modernization_session_steps`: `anchor` arrives aliased exactly as in `modernization_plan` (pass-through, same run); `step_display` is name-free by construction
- `replica_set`: `templates_inherits_from[]`, `templates_acts_as_master[]` (their `*_count` siblings are honest distinct-name counts matching the arrays), `most_similar_template_name_*[]`, `template_similarity_ranking_list[]` (ranking order preserved; only name components aliased)
- `similarity_templates`: `templates[]`

Rules the Companion must respect:

- **Aliases renumber on every ETL run** (deliberately). Never cache or compare aliases across days; never try to de-alias, and never try to recover masked names through other views.
- **Within one run they are consistent across all views** — `Master 292` in a vector anchor is the same master as `Master 292` in the plan and worklist, so pair-joins (`session_kind`, anchor) work normally.
- `Master ?` is a fail-closed sentinel for a name the current catalog can no longer resolve (rare, self-heals next run). `*** No Template Found ***` keeps its normal meaning.
- Display suffixes stay truthful: `(standard template)` / `(standard master)` certify membership in the public HCL list only — so an aliased name can read "Master 12 (standard template)": the CLASS is public, the name is not. Customer-known-only matches carry no suffix.

## Lineage dossier identity (ETL ADRs 0002/0003/0005)

In `modernization_lineage_dossier`, design-document identity is masked unless ALL of the master's instances are trial-selected — which is practically never, so on trial expect it always masked:
- `design_document_name` becomes one ordinal alias per document, `<design_document_type> <n>` (e.g. `Form 7`), numbered within (master, type). It is the same on both row kinds, so element and finding rows of one document still join on (`tmpl_name`, `design_document_type`, `design_document_name`). Like `Master n`, it is valid within one ETL run only. A finding row whose document could not be resolved has no alias: its `design_document_name` is NULL.
- `ddoc_id` and `code_element_name` become NULL (fail closed); `design_document_type` stays visible.
- `detail_masked = true` marks such rows.

Counts, delta classes and verdicts stay honest. As everywhere: masked = "detail withheld — a full license unlocks it", never "no work" and never "no customizations".

## MCP tool surfacing (2026-08 MCP build)

From the 2026-08 MCP server build, the two modernization consumer tools surface masking directly — trust their envelope over re-deriving:

- `modernization_for_app` returns `vector.detail_masked` (this row) and top-level `detail_masking_active` (this ruleset contains masked rows). Masked rows carry a note with the mandated framing; **trial-selected** apps (`detail_masked = false` while `detail_masking_active = true`) carry an alias-only note: their item detail is real, but template names may still be `Master n` aliases.
- `modernization_session_dossier` returns the same top-level `detail_masking_active` plus the masked-detail note when true.
- `detail_masking_active = false` is NOT proof of a full license (edge: an entirely trial-selected kept cohort) — it only certifies that no vector row in the ruleset is masked.
- `modernization_for_app` also reports `vector.n_dismissed_items` (unrelated to trial; see [[schema/tables/modernization-vector]]).

Older MCP builds pass the data through without these fields or notes — the column-level rules on this page then apply directly.

## Availability

Ships with the **2026-08 ETL revision**. From that revision, `replica_set.is_selected_trial_db` and `modernization_vector.detail_masked` exist on ALL license types (appended last; `SELECT *` consumers see two new columns; on full license they are FALSE). Instances on older builds lack the columns, and older TRIAL builds show a false all-clear instead (every vector row `ready`, empty plan/worklist) — if a trial instance shows all-ready with zero items, suspect an old build, not a clean estate.
