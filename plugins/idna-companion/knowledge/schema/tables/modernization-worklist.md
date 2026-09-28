---
id: schema/tables/modernization-worklist
title: cqx_data.modernization_worklist (+ _by_session)
type: schema
status: verified
tags: [domain/idna, topic/modernization, topic/64bit, topic/nomad]
related:
  - concepts/about-modernization-plan
  - concepts/about-modernization-vector
  - schema/tables/modernization-plan
  - schema/tables/modernization-vector
  - schema/tables/code-source
  - schema/topics/trial-license-masking
  - schema/tables/modernization-lineage-dossier
  - schema/tables/modernization-session-steps
  - concepts/about-template-lineage
  - howtos/query-modernization-views
requires_capability: none
source: ETL cqx_data_ifa/010_cqx_data.initialize_components_structure.pxsql (views) over dda_dm.modernization_worklist (ETL script 288); types verified against live instance 2026-07-23; trial masking gated live 2026-08-18; verdict-aware work kinds, inherited_standard scope, reach identity (ETL ADR 0003) and step_kind (ETL ADR 0004) checked live 2026-09-24
last_reviewed: 2026-09-24
---

# cqx_data.modernization_worklist (+ `_by_session`)

## Purpose

**The block-grain home of ALL coding work** — one row per (ruleset, code block) needing a fix. It unifies the "fix once, reapply" patches, per-app solo piece-work, and the anchor masters' own finding blocks (the real fix locations behind every covered item). Ranked **reach-first**. Its companion view `cqx_data.modernization_worklist_by_session` projects the same blocks per session home — the **dossier**: "print me session X's worksheet".

## `modernization_worklist` columns

| Column | Type | Meaning |
|---|---|---|
| `ruleset_id`, `ruleset_name` | text | The compatibility target. **Filter on it.** |
| `code_hash`, `code_hash_id` | text, bigint | The block's identity; `code_hash_id` joins to `code_source` / `find_code_block_usage`. |
| `item_display` | text | What the block is and where it lives — quote it. |
| `item_rank` | integer | Reach-first ranking: reach → `n_masters_containing` → severity. |
| `scope` | text | Work topology: `session` (all occurrences inside ONE session home), `cross_session` (author once, apply in several sittings), or `inherited_standard` (no sitting owns the block — template upgrades/rebases resolve it; `authored_in` is NULL; ranked after all authorable blocks). **NOT the reach split** — see caveats. |
| `reach` | integer | Kept apps carrying the block (this ruleset). |
| `n_cleared_by_refresh` | integer | Occurrences the master session clears — no hand work. |
| `n_persists_after_update` | integer | Covered occurrences whose block SURVIVES the template update (re-applied element, HCL-owned code, unknown fate) — counted as persists, not cleared. |
| `n_sessions` | integer | Session homes touching this block (counts `(session_kind, anchor)` PAIRS). |
| `sessions` | jsonb | Ordered home objects: `{anchor, session_kind, session_rank, is_master_target, has_apply, has_review, requires_master_scan, is_authoring_home, block_verdict}` — authoring home first, then definite-work homes. `block_verdict` is the lineage dossier verdict at that home (NULL outside lineage sessions). |
| `session_anchors` | text[] | Flat anchor list (display convenience; identity is the pair — see caveats). |
| `authored_in`, `authored_session_kind` | text | The authoring home: where the fix is written once; everything downstream is an apply. NULL for `inherited_standard` blocks. |
| `master_targets` | text[] | Masters that carry this block as their own fix location. |
| `apply_replicaids` | text[] | Apps where the authored fix is applied (surplus under scanned context; everything on direct/unify/investigate apps). |
| `review_replicaids` | text[] | Apps with a DIVERGING variant — merge decision, not a paste. |
| `scan_first_replicaids` | text[] | Occurrences under an UNSCANNED master — coverage unknown (see caveats). |
| `severity_weight` | smallint | Numeric severity (rank tie-breaker). |
| `n_masters_containing` | integer | Masters carrying the block (leverage signal). |

**Accounting identity on every row (ETL ADR 0003):** `reach = n_cleared_by_refresh + n_persists_after_update + |apply| + |review| + |scan_first|`. Use it to reconcile any doubted number. (The earlier identity without `n_persists_after_update` is superseded.)

## `modernization_worklist_by_session` (the dossier)

One row per (session home, block): `ruleset_id`, `ruleset_name`, `session_anchor`, `session_kind`, `code_hash`, `code_hash_id`, `item_display`, `item_rank`, `scope`, **`work_kind`**, `block_verdict`, `has_review`, `authored_in`, `authored_session_kind`, `reach`, `severity_weight`, and — appended last (ETL ADR 0004) — **`step_kind`**.

`work_kind` per row (ETL ADR 0003):

- `author` — the fix is written here (or the master target is fixed here). Rebase masters author only custom blocks; upgrade masters never author.
- `apply` — paste a fix authored elsewhere; `authored_in` says where it comes from
- `review` — diverging variant: a merge decision, not a paste
- `scan_first` — master design not collected; the only defensible action is scanning the master
- `cleared_by_update` — informational: the template upgrade/rebase removes this block, no hand work
- `verify_update` — a re-applied or unknown-fate block: check it after the update
- `hcl_owned` — persists in the current HCL template version: HCL's to maintain (or a curation candidate), no code fix to author

`step_kind` groups the row under its step in [[schema/tables/modernization-session-steps]]:
- investigate/unify sessions: every row → `investigate`;
- `scan_first` → `scan_master`;
- `author` → `clear_blocks`;
- `apply` / `review` → `clear_blocks` under direct remediation, `fix_app_copies` (after the refresh) under template kinds;
- `cleared_by_update` → `verify_cleared`;
- `hcl_owned` → `review_hcl_notes`;
- `verify_update` → `verify_carryover` under rebase, `rescan_verify` elsewhere.

```sql
-- Session X's complete worksheet, grouped by procedure step
SELECT step_kind, work_kind, item_display, authored_in, reach
FROM cqx_data.modernization_worklist_by_session
WHERE ruleset_id = '<ruleset_id>'
  AND session_kind = '<session_kind>' AND session_anchor = '<anchor>'
ORDER BY step_kind, work_kind, item_rank
```

For the step ORDER, join [[schema/tables/modernization-session-steps]] on the session key and `step_kind`, and order by `step_rank`.

The dossier is a **worksheet for a human in Domino Designer** — iDNA locates and quantifies the work; the fixing happens outside iDNA. **Say this when presenting a dossier** — a dossier answer must state that the listed work is executed by the customer's developer in Domino Designer, not by iDNA or the Companion. `scan_first` rows translate to a Domino-admin ask: "a template master exists but was not found in any scan — place it on a Domino server iDNA scans."

## Typical use

Biggest fix-once-reapply payoffs:
```sql
SELECT item_rank, left(code_hash, 12) AS hash, item_display, scope, reach,
       n_cleared_by_refresh, authored_in
FROM cqx_data.modernization_worklist
WHERE ruleset_id = '<ruleset_id>'
ORDER BY item_rank
LIMIT 20
```

## Notes and caveats

- **`scope` ≠ reach split.** `cross_session` means the block spans session HOMES; reach means it sits on several APPS. A reach-3 block whose three carriers form one unify session is `scope = 'session'`. Never explain one with the other.
- **Session-home identity is the PAIR `(session_kind, anchor)`**, not the anchor text: an estate master name can simultaneously be a master-track anchor and an investigate anchor — two distinct homes; `n_sessions` counts pairs. Never join or filter on anchor text alone.
- **`scan_first` is not work.** Coverage under an unscanned master is unknown — presenting those occurrences as fixable work (or as cleared) is wrong in both directions. The identity keeps the books honest.
- **`cleared_by_update` and `hcl_owned` rows are not work either.** On rebase/upgrade sessions most block rows are informational; the session's real work is the template operation plus re-applying customizations (see [[concepts/about-template-lineage]]). Never present them as fixes to author.
- **Worklist `reach` counts KEPT apps only.** The vector's shared/solo counts use estate-wide reach across ALL focus apps *including sunset* — two deliberately different scopes; don't reconcile one against the other without accounting for sunset carriers.
- Ranking is reach-first by design (2026-07-09): the earlier masters-first order let blocks in dormant template variants outrank high-payoff blocks.
- TRIAL licenses (2026-08 ETL revision): master anchors in `sessions`/`session_anchors`/`authored_in`/`master_targets` (and `session_anchor` in the dossier view) are per-run `Master n` aliases — pair-joins keep working within a run; `item_display` document names show real only for trial-selected repsets, ordinal aliases otherwise. Counts, reach, and the accounting identity stay honest. See [[schema/topics/trial-license-masking]].
