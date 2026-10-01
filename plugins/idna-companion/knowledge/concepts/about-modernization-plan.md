---
id: concepts/about-modernization-plan
title: Modernization Plan (Sessions and the Worklist)
type: concept
status: verified
tags: [domain/idna, topic/modernization, topic/64bit, topic/nomad]
related:
  - schema/tables/modernization-plan
  - schema/tables/modernization-worklist
  - concepts/about-modernization-vector
  - concepts/design-families
  - concepts/about-template-lineage
  - schema/tables/modernization-session-steps
  - howtos/query-modernization-views
  - schema/tables/code-block-hcl-origin
requires_capability: none
source: ETL dda_dm/scheduled/286 (plan) + 288 (worklist) + 289 (session steps) + ADR 0001 amendments 2026-07-10/11, ADR 0002, 0003, 0004, 0006 (HCL-origin classification) (iDNA_Applications_ETL docs/decisions) + product-owner design dialogue 2026-07-08
last_reviewed: 2026-10-01
---

# Modernization Plan (Sessions and the Worklist)

## The plan-first idea

The naive presentation of modernization readiness is a wall of findings per app. What a customer actually needs sounds like this:

> "Fix template XXX first — that solves 60% of your databases. These 30 similar-but-unlinked apps should become one new template. These 78 shared code blocks clear another 13%. What's left is 56 unique blocks in 7 apps."

That is a **work plan, not a scoring**. iDNA computes it plan-first: the primary artifact is an **ordered queue of work sessions** (`cqx_data.modernization_plan`); the per-app [[concepts/about-modernization-vector]] is each app's projection of that queue ("which session makes *me* green"). Keep this hierarchy in answers: the plan is the product, the vector is the index into it.

## Why the unit of work is a human session

There is **no lossless DXL round-trip** for Notes design elements — you cannot programmatically rewrite a design and trust the result. Every fix ultimately means **a human in Domino Designer with one design container open**. That single constraint generates the whole model:

- **Session** = one design container fully fixed per sitting — a master template, a design-family representative, or a standalone app. You don't open a container twice if you can help it; fix everything in it while you're there.
- **Worklist block ("patch")** = the complement: the *same code block* (identical hash) recurring across containers. The fix is authored once and *reapplied* per occurrence — cheap per repetition.
- KPIs are session-shaped: what does *this sitting* buy the estate?

**iDNA locates and quantifies this work; the work itself happens outside iDNA** — in Domino Designer, by the customer's developer or admin. The Companion helps realize the opportunity and prepares the worksheet; it never executes fixes.

## Sessions (`cqx_data.modernization_plan`)

One row per session per ruleset ([[schema/tables/modernization-plan]]). Session kinds mirror the vector routes: `template/remediate_master`, `template/rebase_ancestor`, `template/upgrade_standard`, `template/unify_similar`, `template/investigate_inheritance`, `direct/remediate_code`. Apps that are `ready` or `ready/with_notes` are in no session — the plan never lists them. The anchor is the container (master template name, design family, or the app itself); `item_display` is the ETL-assembled human sentence — quote it. Lineage sessions read, for example, "Rebase master X onto Y lineage (97% similar)", "Rebase master X: disconnected from master Y, identical code base - upgrade directly", or "Standard master X is already the current HCL version - remaining findings are HCL-owned or curation candidates" (see [[concepts/about-template-lineage]]). On trial licenses, master anchors appear as per-run `Master n` aliases (see [[schema/topics/trial-license-masking]]).

**The session KPI is apps cleared** (`n_apps_green`): how many apps this one session turns fully green — that is the queue's ranking driver (owner decision; no composite scores, the ranking stays explainable). Companions `uad_90d_cleared`/`uad_365d_cleared` express the same thing usage-weighted, for breaking ties. `n_apps_partial` counts apps the session helps but does not clear (they carry surplus/diverging work beyond it).

"Green" is decided per session kind (ETL ADR 0003):
- **`remediate_master`:** the master is scanned and covers every fix item.
- **`upgrade_standard` / `rebase_ancestor`:** the master is scanned and cleared-by-update plus HCL-owned notes account for every fix item (`n_cleared_after_update + n_hcl_notes = n_fix_items`; on these routes `n_hcl_notes` is the lineage-verdict count). Notes re-label green; they do not block it.
- **`direct/remediate_code`:** always green; the session is the app itself.
- **`investigate_inheritance` / `unify_similar`:** never green on their own. They wait on a human decision, so their `n_apps` is potential.

## The worklist (`cqx_data.modernization_worklist`)

Every code fix at block grain, per ruleset ([[schema/tables/modernization-worklist]]) — the former "patches" plus all session piece-work in one place, ranked **reach-first** (kept apps carrying the block, then masters containing it, then severity). Its per-session projection `modernization_worklist_by_session` is the **dossier**: "print me session X's worksheet". Every block carries its work kind, and on apply rows `authored_in` (where the fix comes from). The work kinds (ETL ADR 0003):
- **Work to do:** `author`, `apply`, `review`, `scan_first`.
- **Informational rows of lineage sessions:** `cleared_by_update` (the update removes the block — no hand work), `verify_update` (re-applied or unknown-fate block — check after the update).
- **Notes in every session kind:** `hcl_owned` — the block's exact code is in the current shipped HCL template version: inherited in lineage master homes (the dossier verdict), a verbatim copy in every other home (ETL ADR 0006). HCL's to own, no code fix to author; a copy changes only when it is re-copied. Each such block carries `hcl_origin`, the name-free origin note (template family, Notes versions, current catalog version) — quote it when the user asks where the code comes from.

Blocks no sitting owns have no `authored_in`: `scope = 'inherited_standard'` (resolved entirely by template upgrades/rebases) or `scope = 'hcl_owned'` (the code is current HCL template code; any diverging copies still to review are named in `item_display`). Occurrences of current HCL code outside lineage master coverage count in the worklist identity as `n_hcl_current`, never as work.

## Session steps: the order of work

The plan says which sessions pay off; the worklist says which blocks; **[[schema/tables/modernization-session-steps]] says in what order to do it** — including the template operations that have no block rows (back up the master, replace its design, re-apply customizations, refresh member apps). Every work row groups under its step via the by-session view's `step_kind`. Two ordering rules:
- Master-side work comes before the refresh of member apps.
- App-side fixes come after it, because a refresh overwrites diverging app code. A merge decision exists only where the dossier has review rows; decide it before the refresh.

Steps are guidance only: no completion state.

## Answer shape for "how ready are we" (executive glance)

Lead with the KPI (n ready of n kept focus apps, target named — `ready/with_notes` apps count as ready), name the biggest multiplier sessions next, keep detail at route level — and **always close by offering the drill-down** (full route lists, a session dossier, the block worklist). The offer is part of the answer shape, not optional: an executive glance that just ends strands the user one step short of the plan.

## Answer shape for "what do I do in this session?"

Lead with the procedure — the session's ordered steps — and put the block work beneath the step it belongs to. For lineage sessions most block rows are informational (`cleared_by_update`, `verify_update`, `hcl_owned`); the real work is the template operation plus re-applying the customizations. The verify_update rows are checks after the update, not bookkeeping: name each block. `hcl_owned` rows also appear outside lineage sessions — under `review_hcl_notes` in direct and remediate_master sessions, under the single `investigate` step in investigate / unify sessions (that step's `n_items` excludes them): present them as notes with their origin, never as work. Close with the outside-iDNA principle: the customer's developer or admin executes the steps in Domino Designer / Domino administration.

## The consulting narrative (presentation order)

The computed rankings stay as-is (sessions by apps cleared, blocks by reach), but a consultant narrates them in three tiers:

1. **Custom-master multipliers** (`remediate_master`): fix the master, a refresh clears the cohort. Broken or lost template relations (`unify_similar`, `investigate_inheritance`) belong here as the secondary opportunity.
2. **Standard-template multipliers** (`upgrade_standard`, `rebase_ancestor`): "your teamrooms sit on Domino 11 templates — port them to the current HCL version, where the fixes are already made and HCL maintains them." For detached forks of a standard: "rebase your mail master onto the current HCL mail template and re-apply your customizations — most findings disappear with the update."
3. **Piece-work from the worklist**, heavy-impact first: shared script libraries (high-reach cross-session blocks) down to unique solo blocks.

Customer estates are usually better template-structured than small test environments — expect the multiplier tiers to carry MORE of the work at customers, not less.

## Human decision points (advisory honesty)

The plan's unify/investigate stages assume decisions resolve; real engagements include judgment the data cannot make:

- **Unification** is developer enablement: the pitch is the multiplier ("instead of fixing 10 DBs, unify them under one new shared master — the clean Notes/Domino way"); the material is the family's shared/diverging block sets.
- **Investigate inheritance** means locating the drift (diverging blocks, design similarity) and then a **human reconcilability judgment**.
- **Partial outcomes are legitimate exits:** a 10-DB cohort may end up under two masters; "not reconcilable" is a valid outcome, not a failure.
- **Unscanned masters** are fixed outside iDNA: ask the admin to place the template on a scanned Domino server; until then the plan marks the session `master_unscanned` and the affected blocks `scan_first`.
- **A design replace can destroy fork-only elements.** Rebase sessions include "re-apply customizations" and "verify carryover" steps for exactly that reason — never present a rebase as "just upgrade the template".

Stay advisory on sequencing (the ranking has a stated rationale; staffing and politics are the customer's), but keep risk statements firm — a refresh on a session with diverging blocks needs the review-first warning every time.
