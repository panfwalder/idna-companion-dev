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
  - schema/tables/design-insight-overview-byrepset
  - schema/tables/code-source
  - schema/topics/template-inheritance
  - schema/topics/similarity
  - concepts/about-modernization-vector
  - howtos/use-source-code-remediation-mcp-tools
requires_capability: none
source: ETL cqx_data_ifa/010_cqx_data.initialize_components_structure.pxsql + dda_dm scripts 280/285/286/288; behavior gated live on dev instances 2026-08-17/18 (ETL repo, plans modernization-trial-license + modernization-trial-refinement); template-lineage views per ETL 010 view definitions and ADRs 0002-0004 (2026-09-24); lineage dossier document grain per ETL ADR 0005 (2026-09-25); element pair = the code inside the document per ETL ADR 0008 (2026-10-01); the ETL ADR 0010 element names (Java file base names, host events; trial alias Java n) 2026-10-02; the trial findings surface per ETL ADR 0011 hand-off (trial-findings-surface, final contract 2026-10-05) and the MCP trial flag per Companion ADR 0006 rev 6.8 (2026-10-05)
last_reviewed: 2026-10-05
---

# Trial-license masking across cqx_data

## What trial mode is

iDNA instances on an **evaluation/trial license** (`ia_eval_license` instance property; an ABSENT key also means trial) pseudonymize customer-identifying data across `cqx_data`. Full-license instances are entirely unaffected by everything on this page except two appended columns (see Availability).

## Identity pseudonymization

- `replicaid` values are deterministic 16-hex pseudonyms (`md5` of the raw id) — stable across runs, so joins between views keep working. The pseudonym is NEVER unmasked, not even for trial-selected replica sets.
- Titles read `*** Trial License ***`; `title_unique` reads `*** Trial license (<pseudonym>) ***`; replica paths and mail-in names are masked — EXCEPT fully trial-selected replica sets (M9 revision): they show their real `title`, their LICENSED instances' real `replica_paths`, and a real `title_unique` with the pseudonym in the bracket (`AutoMover [<pseudonym>]`). `mailin_names` and the newest/oldest-design columns stay masked even for selected repsets.

## Honest compute + detail masking (2026-08 ETL revision)

Modernization outputs are **computed on real data** — routing values, session structure, all counts, reach, and KPIs are honest. What is masked is *identity detail*:

- In `modernization_vector.items` / `dismissed_items`, the fields `name` / `element_name` become per-app ordinal aliases (the type plus a per-run ordinal: `name` e.g. `Form 2` or `Scriptlibrary 1`, `element_name` e.g. `Code 3` or `Field 1`; since ETL ADR 0010 (2026-10-02) `Java n` for Java code, and host ordinals can shift per run where one host carries several events) and `noteid` / `designer_link` are null — UNLESS the app's replica set is **trial-selected**. `code_hash_id`, `code_hash`, `insight_id`, `finding_name`, `clearance` stay populated everywhere.
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

## The trial flag (`license`)

MCP builds with the trial flag (2026-10) add `license: "trial"` to `get_idna_session` and to every tool envelope when `cqx_data.database_basic` holds an unlicensed database (`EXISTS (SELECT 1 FROM cqx_data.database_basic WHERE NOT is_licensed)`). The ETL licenses every database on a full license and never a mail or orphan database on trial, so the probe establishes a trial.

- **Absent means unknown, never a full license.** The field is also absent on a small trial whose focus databases (at most 50) are all selected and that has no mail or orphan database, on an empty catalog (before the first ETL run), when the probe fails, and on older MCP builds. Positive masking evidence (`detail_masked`, `*** Trial License ***` titles, `Master n` aliases) still governs how detail is read when the field is absent.
- **It reflects the last completed ETL run.** After a license change it lags one nightly run. That lag is separate from the session's 2-minute TTL and the MCP server's 60-second metadata cache.
- **It is not an authorization signal** and is independent of the access profile: `restricted` / `privileged` decide which tables the MCP may read, the license decides what the data masks. A restricted profile on a full license masks nothing.
- On trial, the finding-surface and lifecycle tools carry one trial note; `findings_for_app` adds a masked-app note for an app with masked rows; `qualify_app_analysis` adds a verdict note when `keep_no_action` rests on a zero finding count; `get_code_block` NOT_FOUND carries a trial hint.

## The finding surface on trial (`design_insight_overview_byrepset`)

What the finding views return on trial depends on the iDNA build and the completed ETL run:

- **Older trial builds:** [[schema/tables/design-insight-overview-byrepset]] returns no rows, and the insights summary counts read 0 for every app. Every finding-based tool then returns zero; that zero does not establish "no findings", and the estate's finding exposure is unavailable. Kept apps that still show not-ready verdicts or active blocks for the same target in the modernization tools, next to a zero finding count for that target, show this case. The modernization tools still provide kept-app readiness and active-block counts after suppressions; they are not estate-wide finding totals.
- **Builds with the trial findings surface:** every focus replica set returns its finding rows, so every count covers the whole estate — the same numbers a full license shows. Fully trial-selected replica sets return detail rows; every other focus replica set returns masked rows.
  - **Row rule:** `database_id_design_reference IS NULL` = masked; set = full detail. On a full license it is never NULL. The MCP tools expose it as `detail_masked`; never infer it from `designer_link`, titles or `replica_set.is_selected_trial_db`.
  - **Masked rows:** `title` reads `*** Trial License ***` (shared by the masked non-selected apps) and `title_unique` `*** Trial license (<pseudonym>) ***`; `design_document_name` and `code_element_name` are aliases `<type> <n>` (`Form 2`, `Code 3`, `Java 1`); `notes_noteid` and `designer_link` are NULL. Ruleset, finding, severity, both types, `code_hash_id` and `source_code_hash` stay real. NULL names stay NULL.
  - **Right after the ETL change is deployed,** before its first completed nightly run, every app is masked; the selected apps then show their real title on masked rows. A masked row can therefore carry a real application title.
- **Alias scope:** `n` is per (application, ruleset, type) within one ETL run — the same alias the document carries in that app's `modernization_vector.items` for that ruleset. A masked location is (`replicaid`, `ruleset_id`, `design_document_type`, alias) within one completed run. The same alias text in two rulesets or two apps can name different documents: never merge masked locations across rulesets or apps, and never persist or compare aliases across runs.
- **Identity:** `replicaid` always. `title_unique` is display only and can be NULL until the next nightly run after a metadata edit in the iDNA UI.
- **Code blocks:** `code_hash_id` is run-local (renumbered nightly); `source_code_hash` is the stable key. [[schema/tables/code-source]] keeps only blocks with a licensed holder, so a block with no licensed holder (held only by non-selected databases) has no source on trial.
- **Templates:** template names are `Master n` aliases (see Master-name aliasing above), so template-grouped counts (`templates_by_downstream_exposure`) include masked contributions under aliased names.

## Counting units

Three units, one definition across tools — label the unit, and never sum or substitute one for another:

- **Occurrence** = one byrepset row (rule x code block x location). `exposure_summary`, `most_affected_apps` and `templates_by_downstream_exposure` totals and severity counts, `findings_for_app` rows, and `get_code_block.finding_occurrences` count occurrences. `affected_applications` counts distinct applications.
- **Finding** = a distinct rule x code block pair within one application and target. The insights summary views' `count_findings` and severity columns count findings (the lifecycle tools read these); their `vulnerability_score` is severity-weighted, not a count.
- **Block** = a distinct code block per kept app after suppressions: the modernization counts (`n_fix_items`, active blocks).

`exposure_summary` includes Exceptional and Insignificant in its total but has no separate buckets for them. Lifecycle summary counts use rule x block pairs, with their documented severity and exclusion rules. AI assistants: do not sum or substitute these counts, occurrence counts and modernization block counts.

## Availability

Ships with the **2026-08 ETL revision**. From that revision, `replica_set.is_selected_trial_db` and `modernization_vector.detail_masked` exist on ALL license types (appended last; `SELECT *` consumers see two new columns; on full license they are FALSE). Instances on older builds lack the columns, and older TRIAL builds show a false all-clear instead (every vector row `ready`, empty plan/worklist) — if a trial instance shows all-ready with zero items, suspect an old build, not a clean estate.
