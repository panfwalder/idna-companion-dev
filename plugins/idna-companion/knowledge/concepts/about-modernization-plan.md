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
  - howtos/query-modernization-views
requires_capability: none
source: ETL dda_dm/scheduled/286 (plan) + 288 (worklist) + ADR 0001 amendments 2026-07-10/11 (iDNA_Applications_ETL docs/decisions) + product-owner design dialogue 2026-07-08
last_reviewed: 2026-07-23
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

One row per session per ruleset ([[schema/tables/modernization-plan]]). Session kinds mirror the vector routes: `template/remediate_master`, `template/upgrade_standard`, `template/unify_similar`, `template/investigate_inheritance/*`, `direct/remediate_code`. The anchor is the container (master template name, design family, or the app itself); `item_display` is the ETL-assembled human sentence — quote it.

**The session KPI is apps cleared** (`n_apps_green`): how many apps this one session turns fully green — that is the queue's ranking driver (owner decision; no composite scores, the ranking stays explainable). Companions `uad_90d_cleared`/`uad_365d_cleared` express the same thing usage-weighted, for breaking ties. `n_apps_partial` counts apps the session helps but does not clear (they carry surplus/diverging work beyond it).

## The worklist (`cqx_data.modernization_worklist`)

Every code fix at block grain, per ruleset ([[schema/tables/modernization-worklist]]) — the former "patches" plus all session piece-work in one place, ranked **reach-first** (kept apps carrying the block, then masters containing it, then severity). Its per-session projection `modernization_worklist_by_session` is the **dossier**: "print me session X's worksheet" — every block with its work kind (`author` / `apply` / `review` / `scan_first`) and, on apply rows, `authored_in` (where the fix comes from).

## Answer shape for "how ready are we" (executive glance)

Lead with the KPI (n ready of n kept focus apps, target named), name the biggest multiplier sessions next, keep detail at route level — and **always close by offering the drill-down** (full route lists, a session dossier, the block worklist). The offer is part of the answer shape, not optional: an executive glance that just ends strands the user one step short of the plan.

## The consulting narrative (presentation order)

The computed rankings stay as-is (sessions by apps cleared, blocks by reach), but a consultant narrates them in three tiers:

1. **Custom-master multipliers** (`remediate_master`): fix the master, a refresh clears the cohort. Broken or lost template relations (`unify_similar`, `investigate_inheritance`) belong here as the secondary opportunity.
2. **Standard-template multipliers** (`upgrade_standard`): "your teamrooms sit on Domino 11 templates — port them to the current HCL version, where the fixes are already made and HCL maintains them."
3. **Piece-work from the worklist**, heavy-impact first: shared script libraries (high-reach cross-session blocks) down to unique solo blocks.

Customer estates are usually better template-structured than small test environments — expect the multiplier tiers to carry MORE of the work at customers, not less.

## Human decision points (advisory honesty)

The plan's unify/investigate stages assume decisions resolve; real engagements include judgment the data cannot make:

- **Unification** is developer enablement: the pitch is the multiplier ("instead of fixing 10 DBs, unify them under one new shared master — the clean Notes/Domino way"); the material is the family's shared/diverging block sets.
- **Investigate inheritance** means locating the drift (diverging blocks, design similarity) and then a **human reconcilability judgment**.
- **Partial outcomes are legitimate exits:** a 10-DB cohort may end up under two masters; "not reconcilable" is a valid outcome, not a failure.
- **Unscanned masters** are fixed outside iDNA: ask the admin to place the template on a scanned Domino server; until then the plan marks the session `master_unscanned` and the affected blocks `scan_first`.

Stay advisory on sequencing (the ranking has a stated rationale; staffing and politics are the customer's), but keep risk statements firm — a refresh on a session with diverging blocks needs the review-first warning every time.
