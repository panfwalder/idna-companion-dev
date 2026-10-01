---
id: schema/tables/modernization-session-steps
title: cqx_data.modernization_session_steps
type: schema
status: verified
tags: [domain/idna, topic/modernization, topic/64bit, topic/nomad]
related:
  - schema/tables/modernization-plan
  - schema/tables/modernization-worklist
  - schema/tables/modernization-lineage-dossier
  - schema/tables/template-ancestry
  - schema/topics/trial-license-masking
  - concepts/about-modernization-plan
  - howtos/query-modernization-views
requires_capability: none
source: ETL cqx_data_ifa/010 (view + COMMENT) over dda_dm.modernization_session_steps (create 031, ETL script 289); ETL ADR 0004; column types verified against the live instance 2026-09-24; HCL-origin step changes (ETL ADR 0006: review_hcl_notes in direct / remediate_master, the verify counts, the investigate exclusion) checked live 2026-10-01
last_reviewed: 2026-10-01
---

# cqx_data.modernization_session_steps

## Purpose

**The ordered procedure for every modernization session** — what a user does, in order, to come out with clean databases (ETL ADR 0004). The plan says WHICH sessions pay off; the worklist says WHICH blocks; the steps say IN WHAT ORDER to do the work — including the template operations (backup, replace design, re-apply customizations, refresh member apps) that have no block rows.

**Guidance only.** Steps carry no completion state, never block anything, and never instruct editing standard code. They are rebuilt on every ETL run from the current analysis.

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `ruleset_id`, `ruleset_name` | text | The compatibility target. **Filter on it.** |
| `session_kind`, `anchor` | text | The plan session — identity is the PAIR, as in [[schema/tables/modernization-plan]]. |
| `step_rank` | integer | 1..n, contiguous per session. **Always ORDER BY it.** |
| `step_kind` | text | One of twelve kinds (below). |
| `step_display` | text | Name-free guidance sentence — quote it. |
| `n_items`, `item_source` | integer, text | The step's countable payload: `blocks`, `elements` or `apps`; both NULL when the step has none. |

## Procedure per session kind

Master-side work comes BEFORE the refresh of member apps; app-side work comes AFTER it (a refresh overwrites diverging app code); the final rescan comes last. Conditional steps appear only when they have something to do — never a zero-count step.

| Session kind | Steps in order (conditional in *italics*) |
|---|---|
| `template/rebase_ancestor` | *scan_master* → backup_master → replace_design → reapply_customizations → *clear_blocks* → *verify_carryover* → *verify_cleared* → *review_hcl_notes* → refresh_apps → *fix_app_copies* → rescan_verify |
| `template/upgrade_standard` | *scan_master* → backup_master → *replace_design* (skipped when the master already is the newest version) → *clear_blocks* → *verify_cleared* → *review_hcl_notes* → refresh_apps → *fix_app_copies* → rescan_verify |
| `template/remediate_master` | *scan_master* → backup_master → *clear_blocks* → *review_hcl_notes* → refresh_apps → *fix_app_copies* → rescan_verify |
| `direct/remediate_code` | *clear_blocks* → *review_hcl_notes* → rescan_verify |
| `template/investigate_inheritance`, `template/unify_similar` | investigate (one step; all the session's blocks wait on its outcome) |

Step meanings in brief: `scan_master` — scan the master's design first (the plan may change); `backup_master` — back up the master before touching its design; `replace_design` — replace the master's design with the current version of its lineage; `reapply_customizations` — restore ALL customizations from the backup against the dossier's element list, including finding-free modified elements; `clear_blocks` — author or apply fixes; `verify_carryover` — verify that fork-only elements survived the replace; `verify_cleared` — confirm the findings the update clears are gone; `review_hcl_notes` — review findings in current HCL template code (HCL's to own, nothing to author): in rebase / upgrade sessions it groups both the master's `persists_hcl_owned` verdict rows and app-side verbatim copies, in direct / remediate_master sessions the verbatim copies (ETL ADR 0006); `refresh_apps` — point member apps at the master (Replace Master Template where not yet inherited), then refresh their design; `fix_app_copies` — after the refresh, fix blocks in app copies the master does not cover; `rescan_verify` — re-scan and confirm the remaining findings match the plan.

**Counts that are not the row count (ETL ADR 0006):**

- **Upgrade `rescan_verify`** carries `n_items` / `item_source = 'blocks'` when the session has `verify_update` rows: the step text adds "re-check the M block(s) whose fate after the update is unknown", and `n_items` is M. Without such rows `n_items` stays NULL.
- **Rebase `verify_carryover`** counts design ELEMENTS (`item_source = 'elements'`) when the master has fork-only / all-custom elements; when the session also has `verify_update` blocks, the text ends "Then re-check the M flagged block(s) preserved through the update." and `n_items` stays the element count. Without elements it counts the blocks.
- **`investigate`** groups every row of the session, `hcl_owned` rows included, but its `n_items` counts only the flagged blocks that ride on the outcome; the text names the rest: "(M further block(s) are current HCL template code - no code fix to author)". The session's row count can therefore exceed `n_items` — present those rows as notes.

## Relationships

- Session key (`ruleset_id`, `session_kind`, `anchor`) = [[schema/tables/modernization-plan]]'s session key.
- Work items group under their step through the trailing `step_kind` column of `modernization_worklist_by_session` ([[schema/tables/modernization-worklist]]).
- `reapply_customizations` and `verify_carryover` counts come from the element rows of [[schema/tables/modernization-lineage-dossier]].

## Typical use

```sql
-- The procedure for one session, with its work items beneath each step
SELECT st.step_rank, st.step_kind, st.step_display, st.n_items, st.item_source,
       count(w.code_hash) AS work_rows
FROM cqx_data.modernization_session_steps st
LEFT JOIN cqx_data.modernization_worklist_by_session w
  ON w.ruleset_id = st.ruleset_id AND w.session_kind = st.session_kind
 AND w.session_anchor = st.anchor AND w.step_kind = st.step_kind
WHERE st.ruleset_id = '<ruleset_id>'
  AND st.session_kind = '<session_kind>' AND st.anchor = '<anchor>'
GROUP BY 1, 2, 3, 4, 5
ORDER BY st.step_rank
```

## Notes and caveats

- **Scan order is not a contract** — without `ORDER BY step_rank` the procedure reads in random order.
- **Mode-specific wording is in the display.** For a shipped-reference ancestor (dossier mode B), `verify_carryover` speaks of "all-custom code elements (possibly new or fully rewritten)", not "fork-only elements"; keep that distinction when paraphrasing.
- **The steps describe work the customer's developer or admin executes** in Domino Designer / Domino administration, not iDNA or the Companion.
- TRIAL licenses: `anchor` arrives aliased exactly as in `modernization_plan` (per-run `Master n`), so the pair still joins within a run; `step_display` is name-free by construction. See [[schema/topics/trial-license-masking]].
- Access: granted to the restricted role (ADR 0006 appendix). `modernization_session_dossier` returns a session's steps (`steps`) with its work rows in procedure order, in any profile; on an older MCP build without that field, privileged sessions reach the view via `execute_sql`.
