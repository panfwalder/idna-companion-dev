---
id: howtos/use-source-code-remediation-mcp-tools
title: Use the source-code-remediation MCP tools
type: howto
status: verified
tags: [domain/idna, topic/modernization, audience/consultant]
related:
  - playbooks/source-code-remediation/remediation-routing
  - playbooks/source-code-remediation/assess-exposure
  - playbooks/source-code-remediation/classify-template-paths
  - playbooks/source-code-remediation/classify-shared-code-paths
  - playbooks/source-code-remediation/build-remediation-plan
  - concepts/compatibility-ruleset-routing
  - concepts/notes-64bit-remediation-profile
  - concepts/nomad-web-mobile-remediation-profile
  - schema/tables/design-insight-overview-byrepset
  - schema/tables/code-source
  - playbooks/application-lifecycle/lifecycle-routing
  - schema/topics/trial-license-masking
  - concepts/about-template-lineage
  - schema/tables/code-block-hcl-origin
requires_capability: none
source: mcp-server Phase 2 tool surface and remediation playbook mapping; trial-masking surfacing per the 2026-08 MCP build (docs/plans/completed/trial-masking-mcp-consumption); tool output vs. the ETL template-lineage revision (ETL ADRs 0002-0004) checked against the tool code and live data 2026-09-24; HCL-origin follow-up (ETL ADRs 0006 / 0007; docs/plans/completed/hcl-origin-followup) checked against the tool code and live data 2026-10-01; element pair = the code inside the document per ETL ADR 0008 (2026-10-01); Java file names and host events per ETL ADR 0010, the one-block routing line and the get_code_block notes (docs/plans/active/code-block-eval-fixes) 2026-10-02
last_reviewed: 2026-10-02
---

# Use the source-code-remediation MCP tools

## Question

"Which MCP tool should I use for this remediation question?"

## Approach

When the MCP server is available, prefer the semantic remediation tools over ad hoc SQL. The playbooks remain the "why" and "when" layer; the MCP tools are the execution layer for the most common consultant questions.

Use `execute_sql` only when:

- the active session is **permissive** (`access_profile = 'privileged'`) — `execute_sql` is privileged-only and is refused under restricted mode before any SQL is issued; in restricted mode stay with the semantic tools, or present documented SQL for the customer to run manually with their own credentials
- no existing semantic tool covers the question
- you already know the exact table and column names
- the caller needs a custom slice that is narrower or different from the semantic tool outputs

## Required inputs

- target resolved via [[concepts/compatibility-ruleset-routing]]
- either a known target shortcut such as Notes 64-bit / Nomad Web/Mobile, or an explicit `ruleset_id`
- application identifier (`replica_id`) or `code_hash_id` where relevant

## Routing map

Readiness / work-plan questions (the modernization surfaces):

- "Which apps aren't ready for `<target>`?" / "How ready are we?" / "Where do we start?" -> `modernization_plan_overview` (readiness KPI + session queue ranked by apps cleared)
- "Is app `<X>` ready / what fixes it?" -> `modernization_for_app` (replicaid or exact title)
- "What exactly do I do in session `<anchor>`?" / "In what order do I do this session?" -> `modernization_session_dossier` (session identity is the PAIR kind + anchor); it returns the session's ordered `steps` and the work rows in procedure order
- Template-lineage questions ("which standard template was this master forked from?", "what did we customize?", "what survives the update?") -> `modernization_lineage_overview` (counts only; a master name, or an app on a rebase / upgrade route or a with-notes app whose notes come from a lineage verdict), then drill into one count with `modernization_lineage_dossier` using exactly the values the overview shows. Any profile. On an older MCP build without these tools, privileged sessions query `template_ancestry` and `modernization_lineage_dossier` via `execute_sql` ([[howtos/query-modernization-views]]); restricted sessions answer from [[concepts/about-template-lineage]] and say that lineage detail is not available in this session.
- "Which fixes do we author once and reapply?" -> `modernization_plan_overview` top blocks / the worklist ([[playbooks/source-code-remediation/analyze-shared-vs-solo-blocks]])
- "Which code blocks give the biggest remediation leverage?" -> the BLOCK-grain surfaces: `modernization_plan_overview` top blocks / the worklist ranked by reach ([[playbooks/source-code-remediation/analyze-shared-vs-solo-blocks]]) — NOT `templates_by_downstream_exposure` (template grain). Keep the grain the user asked for: blocks stay blocks even when a template lens is also available; offer the template view as a complement, never as the substitute answer.

Exposure / finding-volume questions:

- "How big is our 64-bit or Nomad problem (findings, severity)?" -> `exposure_summary`
- "Show me the most affected apps." -> `most_affected_apps`
- "Show me the findings in app `<replica_id>`." -> `findings_for_app`
- "What should I fix first in this app?" -> `prioritize_findings`
- "Should our developer fix finding F in document D of app A?" -> `modernization_for_app` for the app's `work_state` of that block (raise `top_n` to the app's block count when the list is truncated; `hcl_owned` blocks rank last) and `get_code_block` on the block (its HCL origin and the shared-vs-copied note); a testing-cue finding is still a test, not a fix ([[concepts/notes-64bit-findings-taxonomy]])
- "Show me the normalized source for code block `<code_hash_id>`." -> `get_code_block`
- "Where else does this code block appear?" -> `find_code_block_usage` (pass `target`/`ruleset_id` for per-ruleset app reach; without it the rows are occurrence-grain across all rulesets)
- "Which template families drive the most downstream exposure?" -> `templates_by_downstream_exposure`

Readiness questions route to the modernization tools, volume/severity questions to the exposure tools — do not answer one with the other.

## Procedure

1. Bind the compatibility target first.
   Use [[concepts/compatibility-ruleset-routing]] to resolve the target profile before choosing a remediation tool.

2. Prefer the narrowest semantic tool that matches the customer question.
   Do not start with `execute_sql` if the question is already covered by a remediation tool.

3. Preserve the consulting scope in the answer.
   State whether the result is Focus applications only, all applications, one app, or one shared code block.

4. Use ad hoc SQL only as a fallback.
   If a semantic tool does not cover the needed cut, explain the gap and then use `execute_sql` with explicit intent.

## Interpretation

- `modernization_plan_overview` IS the customer-facing work plan: quote `item_display` sentences (the queue now includes `template/rebase_ancestor` sessions); sessions rank by `n_apps_green` (apps one sitting turns fully green), not `n_apps`; close overview answers by offering the drill-down. Its top blocks carry `n_hcl_current` (verbatim-copy occurrences of current HCL code, part of the reach identity — notes, never work) and `hcl_origin`.
  - **Readiness KPI:** `n_ready` counts `ready` AND `ready/with_notes` apps (all remaining findings are HCL's to own in the current template version — inherited, or verbatim copies in an app that does not inherit from HCL: there all its active items are current HCL template code and none is a diverging copy of such code (a diverging copy is review work; ETL ADR 0007 D-6) — READY per the ETL product stance); `n_ready_with_notes` is their share, and a note states it when it is above zero. Report "N ready, of which M with notes on HCL-shipped code" — `n_ready_without_notes` is the rest (on an older MCP build without that field: `n_ready - n_ready_with_notes`); never add `n_ready_with_notes` on top of `n_ready` (in a table: ready without notes / ready with notes / not ready sum to the kept total).
  - **If `n_ready_with_notes` is absent** (an older MCP build), `n_ready` counts only apps whose vector is exactly `ready`, and the with-notes apps sit in `n_not_ready`. Privileged sessions: correct the figure with the `ready/%` query in [[howtos/query-modernization-views]]. Restricted sessions: say that the KPI counts only apps with no findings, and that apps whose findings are all HCL's to own (inherited or verbatim copies) are also ready but not broken out by this tool.
- `modernization_for_app` is the per-app verdict + block work list (work states `cleared_by_refresh` / `apply` / `review` / `scan_first`, and `hcl_owned` on every route, plus the update fates below on lineage routes; the authoring home is the pair `authored_session_kind` + `authored_in`; and the block's HCL origin in `hcl_origin`). The labels come from the ETL's per-item occurrence state and verdict (ETL ADR 0007, see [[schema/tables/modernization-vector]]), never from the tool's own reading of coverage. Sunset apps are MIS_ROUTED to `sunset_disposition` — follow it.
  - **`hcl_owned` on any route:** the block's exact code is current HCL template code — inherited (the lineage verdict) or a verbatim copy (every `hcl_current` occurrence without that lineage verdict: a direct, unify or investigate app - also one that merely declares an HCL-named template no estate-operated master backs - and surplus code on a lineage route; e.g. a direct app that copied an HCL script library). HCL's to own, no code fix to author; a copy changes only when it is re-copied. Quote `hcl_origin` (template family, Notes versions, current catalog version) when the user asks whether to fix it or where it comes from. An "Older HCL ..." origin on a work block means "compare with the current HCL version before hand-fixing". When `origin_shortened` is true a note says so; the full sentence is in [[schema/tables/code-block-hcl-origin]].
  - **A NULL `work_state` with the "contract unavailable" note** means the iDNA ETL and this MCP build disagree on the per-item contract (the ETL predates ETL ADR 0007, its deploy is incomplete, or it is newer): quote the route and the display, and never derive block states yourself.
  - **Lineage routes:** on `template/upgrade_standard` and `template/rebase_ancestor` apps, a covered block's `work_state` is its update fate, taken from the item's lineage verdict (the primary master's): `cleared_by_update` (the update removes it), `cleared_by_refresh` (custom code fixed in the master), `hcl_owned` (persists in the current HCL version — HCL's to maintain, nothing to author; a surplus copy of current HCL code reads `hcl_owned` too, as a verbatim copy), `verify_update` (re-applied or unknown fate — check after the update). The vector adds `ancestor_name` / `ancestor_kind` / `ancestor_similarity` on rebase rows, `n_cleared_after_update` on the lineage routes, and `n_hcl_notes` on every row with active items (lineage: the HCL-owned notes by the dossier; elsewhere: the verbatim copies). Still quote the display: "N of M blocks clear with the update … to verify after the update … HCL-owned note(s)"; a direct display may end "; K in current HCL code".
  - A `ready/with_notes` app is READY: every block reads `hcl_owned`. Present them as notes on HCL-shipped code — "maintained by HCL" only for an app that inherits from HCL; for verbatim copies (all its active items are current HCL template code and none is a diverging copy of such code (a diverging copy is review work; ETL ADR 0007 D-6)) say "verbatim copies of current HCL template code, no fix to author" — never as work.
  - **If these fields are absent** (an older MCP build), every covered block reads `cleared_by_refresh`, which on these routes only means "covered by the master". Take the fate from the display, and do not tell the customer every covered block disappears.
- `modernization_session_dossier` is a worksheet executed by the customer's developer in Domino Designer — iDNA locates and quantifies; scan_first rows are a Domino-admin ask.
  - **It returns `steps`, the session's ordered procedure** (`step_rank`, `step_kind`, `step_display`, `n_items`, `item_source`; guidance only, no completion state), **and the rows in procedure order**: every row names its step in `step_kind` and carries `block_verdict` where the lineage verdict applies. Present the worksheet step by step, with each step's rows beneath it.
  - Work kinds: all seven (ETL ADR 0003). `author`, `apply`, `review`, `scan_first` are hand work; `verify_update` (check after the update) and `cleared_by_update` (the update removes the block — no hand work) are informational rows of rebase/upgrade sessions; `hcl_owned` (the block's exact code is current HCL template code — inherited in lineage master homes, a verbatim copy in every other session; HCL's to own, no code fix to author) can appear in any session kind. Never present those three as fixes. verify_update rows are real checks, not bookkeeping: name each block. On a rebase they sit under `verify_carryover`, whose step text counts design elements and, where both exist, adds the re-check blocks; on an upgrade under `rescan_verify`, whose `n_items` is their count.
  - Rows carry `hcl_origin` (the HCL template family and versions the code ships in). In an investigate / unify session the `hcl_owned` rows sit under the single `investigate` step, whose `n_items` excludes them (the step text names them); the tool's note says so — present them as notes.
  - Review rows: a refresh overwrites diverging app code. DECIDE the merge before the refresh: keep the variant by moving it into the master, or accept the master's version. The app-copy fix itself comes after the refresh (`fix_app_copies`).
  - An empty `steps` comes with a note (the steps table is out of sync, e.g. right after an ETL deploy): present the rows by work kind. A truncated dossier names the steps whose rows were cut: offer a higher `top_n`, or the full list in the Modernization Session Work List dashboard (`{{idna_server}}/metabase/dashboard/10000050`, in Metabase; access depends on your Metabase permissions): choose the target and this session.
  - **If `steps` is absent** (an older MCP build), the rows sort by work kind, not by procedure. For a rebase/upgrade session, frame the worksheet with the procedure from [[concepts/about-template-lineage]] (back up → replace design → re-apply customizations → verify → refresh → fix app copies → re-scan).
- `modernization_lineage_overview` and `modernization_lineage_dossier` answer for ONE master's lineage (ETL ADRs 0002–0005):
  - The overview returns COUNTS ONLY: the ancestry header (`master`), `element_counts` per `design_document_type` × `delta_class` (target-independent), `finding_counts` per verdict × rule, and `totals`. Reach it by `master_name`, or by an app (replicaid / title) on a rebase or upgrade route or a with-notes app whose notes come from a lineage verdict; other apps (a with-notes app on verbatim copies included) are MIS_ROUTED to `modernization_for_app`, and a name that anchors a session to `modernization_session_dossier` — follow them.
  - Drill into one count with `modernization_lineage_dossier`, passing exactly the values the overview shows: `design_document_type` + `delta_class` for an element cell, `verdict` + `rule` for a finding cell, and `design_document_name` to open one document. It needs at least one filter.
  - Verdicts are per block: count distinct `code_hash_id` values. A block in several documents repeats its verdict on each and counts once.
  - Type values are single tokens such as `Form`, `Scriptlibrary` or `Sharedfield`; pass them as the overview shows them (case and spaces are ignored anyway).
  - Same-named documents (e.g. two forms `panSetup`) are separate documents, even when every count matches: report them as two, never merge them.
  - A master without any dossier rows is not a clean lineage. For a fork master the ETL did not classify its documents, so its document-level lineage is not available (the ancestry header is still real) — answer from `modernization_for_app` and do not say "nothing customized". For a standard master, the fates are unknown whenever its apps show covered findings. Relay the tool's note.
  - Each finding carries `code_hash_id`: continue with `get_code_block` for the code, `find_code_block_usage` for its spread.
  - On trial licenses the document names are per-document aliases ("Form 7", valid within one ETL run). Present them as withheld detail and never try to de-alias them.
  - Customized cells (modified / added) and verdict × rule cells drill down completely within the limits; a large inherited cell may truncate (`documents_truncated`, with a note carrying the cell total). **The full list** is the Template Lineage dashboard (`{{idna_server}}/metabase/dashboard/10000049`, in Metabase; access depends on your Metabase permissions): its tables download as CSV where downloads are enabled, and the user can share that file with the assistant when the full list is really needed.
  - Relay the tools' notes: mode B wording, mode F (standard master), rename-only, and any cross-check warning (an alias may point at a different master — re-resolve it in this session).
  - **If these tools are absent** (an older MCP build), use the privileged `execute_sql` path in [[howtos/query-modernization-views]]; restricted sessions say that lineage detail is not available.
- The plan overview, for_app and session dossier error with NOT_FOUND + a manual-fallback hint when the ruleset is not in the modernization precompute — that means "use the manual playbook method", not "empty estate". The lineage overview does the same on the app path; on the master path, and in the drill-down, it answers with the element counts (target-independent, real) and a note that the finding verdicts are not precomputed for this ruleset.
- **Trial licenses (2026-08 MCP build):** all modernization tools, the lineage tools included, return trial-aliased names (`Master n` anchors and masters in `modernization_plan_overview` too; per-document aliases in the lineage drill-down). `modernization_for_app` and `modernization_session_dossier` also surface `detail_masking_active` (top-level) and, on for_app, `vector.detail_masked` + `vector.n_dismissed_items`; their notes carry the required framing — masked detail is "withheld, a full license unlocks it", NEVER "no work", and `Master n` aliases renumber every ETL run. Relay those notes; rules in [[schema/topics/trial-license-masking]].
- `exposure_summary` is the aggregate volume view.
- `most_affected_apps` is the named application working set.
- `findings_for_app` is the bridge from application scope into concrete finding rows.
- `prioritize_findings` is the planning lens inside one application.
- `get_code_block` and `find_code_block_usage` are the shared-code drilldown pair. Ruleset-bound `find_code_block_usage` reach is raw-finding reach: an upper bound when FP suppressions are configured — the ignore-aware numbers are `modernization_worklist.reach` and the vector counts.
  - **Shared or copied:** identical code in several apps is shared through one source only when an estate-operated master is shown to contain this exact block (the target-bound worklist's `n_masters_containing` / `master_targets`, for_app's coverage); otherwise plan it as copies, fixed per copy or authored once in the session named by `scope` / `authored_in`. Say which case applies ([[concepts/code-block]]).
  - From the 2026-10 MCP build `get_code_block` also returns `hcl_in_current`, `hcl_origin` (`hcl_in_current` true: current HCL code, the HCL note fires; false with an "Older HCL ..." origin: older-only HCL code, work - compare with the current HCL version before hand-fixing; false with a NULL origin: not shipped HCL code; both NULL only when the origin view is unavailable, which a note says) and `affected_kept_applications`, with an HCL note (current HCL code: no customer code fix to author; "maintained by HCL" only for a covered `persists_hcl_owned` lineage occurrence, a verbatim copy otherwise) and, for a block in several apps, a provenance note (copies, or "not proof of one shared source"; `downstream_templates` counts inherited OR similar templates). Relay both; `affected_applications` spans every ruleset and lifecycle state, so sunset carriers are retirement candidates, not fix targets.
- Finding rows (`findings_for_app`, `find_code_block_usage` occurrence mode, and `prioritize_findings`' `sample_*` columns):
  - The location is the design document (`design_document_type` + `design_document_name`). The code element pair is subordinate detail, never the location. On ETL builds with ADR 0008 it is the code inside the document: `Code` + an opaque DXL identifier, such as an event, LotusScript section, entry sub, sub / function name, or raw `code[1]`, or a host such as `Field` / `Action`. Since ETL ADR 0010 (2026-10-02) also `Java` + the file's base name (not unique across documents: keep the document pair), and a host without a name with its event (`pardef[4] > hidewhen`, `button[1] > click`) — opaque display strings, never split them. Older builds may repeat the document or show `Globals / globals[1]`; the rule holds there too.
  - Say "Script Library SetPrevLib, code element CheckAdminAccess" or "Agent (ProcessThumbs), Java file JavaAgent.java". Never say "in Code options", and do not call the name a sub or function unless the code shows it.
  - `prioritize_findings` is one row per finding x block from the 2026-10 MCP build, with its sample taken from the first finding row in the server's sort order of (document name, element name, document type, element type) (`sample_design_document_type` / `sample_code_element_type`; absent on older builds).
- `templates_by_downstream_exposure` is the template-leverage lens at the finding level.

**Lifecycle direction hint (read-only triage, not a disposition).** **When the MCP returns** `lifecycle_direction_hint` on `most_affected_apps` rows or the `findings_for_app` summary (`keep` / `sunset` / `null`), treat it as the coarse direction of the app's ETL lifecycle vector — surfaced so a remediation conversation can spot apps that may not be worth remediating. It is **not** an actionable disposition: a `sunset` hint means *consider retiring instead of fixing*. Route to the lifecycle tools (`sunset_candidates` / `sunset_disposition`, or `keep_candidates` / `keep_disposition`) for the gated decision — never act on the hint directly. `null` = non-focus or not classified. **If the field is absent** (an older MCP build that predates this field), continue the remediation flow normally and do not infer lifecycle direction. See [[playbooks/application-lifecycle/lifecycle-routing]].

## Fallback rule

If the MCP server is unavailable, fall back to the wiki's documented SQL patterns in the playbooks and schema pages. The playbooks still define the right sequence even when execution falls back to SQL.

## Sources

Phase 2 MCP implementation and remediation playbook mapping.
