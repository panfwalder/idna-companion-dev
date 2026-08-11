---
id: concepts/about-modernization-vector
title: Modernization Vector
type: concept
status: verified
tags: [domain/idna, topic/modernization, topic/64bit, topic/nomad]
related:
  - schema/tables/modernization-vector
  - concepts/about-modernization-plan
  - concepts/design-families
  - concepts/lifecycle-vector
  - concepts/compatibility-ruleset-routing
  - concepts/about-template-inheritance
  - concepts/design-similarity
  - howtos/query-modernization-views
requires_capability: none
source: ETL dda_dm/scheduled/285_dda_dm.modernization_vector.update.sql (routing CASE) + ADR 0001 (iDNA_Applications_ETL docs/decisions) + product-owner design dialogue 2026-07-08
last_reviewed: 2026-07-24
---

# Modernization Vector

## What it is

`modernization_vector` is the **ETL-computed answer to "is this app ready for target X, and if not, what kind of fix gets it there?"** — one value per **(kept focus application × compatibility ruleset)**, exposed on `cqx_data.modernization_vector` (see [[schema/tables/modernization-vector]]) and, for the two built-in targets, as headline columns on `cqx_data.replica_set`.

Customers rarely ask for it by name. The questions it answers sound like *"What apps will break if I move to 64-bit Notes clients?"* or *"Which of my apps aren't ready for Nomad Web?"* — answer those from the vector's readiness split, not from raw finding counts.

Two framing rules for every answer:

- **Findings measure the presence of non-portable code, not guaranteed breakage.** A well-maintained app can legitimately carry findings for runtime-guarded, desktop-only features (the stock HCL mail template does). Say "carries code that needs review/fixing for the target", not "will crash".
- **Readiness is per-target.** The same app can be `ready` for Notes 64-bit and `template/remediate_master` for Nomad. Always name the ruleset an answer applies to.

## The value tree

```
ready
template/remediate_master      [/unscanned_master_template | /diverging_code_blocks | /surplus_code_blocks]
template/upgrade_standard      [same qualifiers]
template/investigate_inheritance/similar_standard
template/investigate_inheritance/similar_master
template/unify_similar
direct/remediate_code
```

`modernization_vector_display` carries the user-facing sentence for each value (e.g. *"Remediate master X: 5 of 8 blocks clear with the master; 2 diverging (refresh would overwrite — review first)"*). Recognize and reuse those words — do not invent parallel phrasing.

## How routing works (and why)

The routing is a **relationship decision tree, not a scoring function** — because that is how a Notes developer triages an estate. Findings decide only *ready vs. not ready*; everything after that is a question about relationships:

1. **No findings for the target → `ready`.** Short-circuits before any relationship logic.
2. **"Who's your master?"** App inherits from an estate-operated master template → `template/remediate_master` — fix the master once, a refresh clears the inheriting apps. Templates carry a Notes estate; *fix multipliers first, then leftovers*. If the primary master's name is standard-recognized → `template/upgrade_standard` instead: a pristine standard is HCL's work — **upgrade it, don't edit it**; the current HCL version already contains the fix.
3. **No master link, but ≥85% design-similar to a classifiable template** → `template/investigate_inheritance` — `/similar_master` (template operated in the estate) or `/similar_standard` (name-recognized only). **Estate-operated beats standard-recognized**: a renamed standard carrying customizations is a customer asset — the customizations are precisely what needs the work.
4. **No template relation, but design-family peers exist** (≥1 kept peer) → `template/unify_similar` — similar-but-unlinked apps are a *missed multiplier*: candidates to unify under one new shared master (see [[concepts/design-families]]). The trigger is the **design relationship alone** — unify does NOT imply the peers carry the same finding hashes (a unify app's blocks can be entirely solo; similarity tolerates code drift, hashes don't).
5. **Otherwise → `direct/remediate_code`** — a genuine one-off.

The primary master is the most-covering scanned template (scanned preferred, alphabetical tiebreak); coverage is **primary-master-scoped** on purpose — the question is "does *my* master's session clear me", not "does any template anywhere contain this hash".

## Qualifiers are risk communication (worst-wins)

The `remediate_master`/`upgrade_standard` qualifiers are warnings a consultant would voice, in escalating order (`unscanned_master_template` > `diverging_code_blocks` > `surplus_code_blocks` — the worst present is displayed):

- **`/unscanned_master_template`** — *coverage unknown*: the master's design was never collected. The first action is a scan, not a fix — ask the Domino admin to place the master on a server iDNA scans. Coverage columns are NULL here (unknown, not zero).
- **`/diverging_code_blocks`** — *data-loss risk*: the app carries a local variant of an element the master also has. **A template refresh overwrites diverging blocks — review them before any refresh.** This warning is firm, always; it is the single most important risk statement in the feature.
- **`/surplus_code_blocks`** — *residual work*: blocks beyond the master's design; the master's session alone won't turn this app fully green.

## Standard-template recognition and the three labels

The recognizer matches template names against the HCL standard set with **suffix tolerance but not prefix tolerance**: `StdR11Blog_ifaV11` still counts as standard; `Pana_StdR4...` does **not** — a rename-with-prefix signals customization. **The three-way label describes the SIMILARITY TARGET named in the display, not the investigating app itself** — in "Investigate inheritance: 99% similar to StdR10Discussion (standard master)", the label classifies StdR10Discussion:

- `(standard master)` — standard-named AND operated in the estate
- `(estate master)` — operated here, custom name
- `(standard template)` — name-recognized only, not operated in this estate

When a similarity target qualifies both ways, **estate-operated beats standard-recognized** — a renamed standard carrying customizations is a customer asset; the customizations are precisely what needs the work (same precedence as in the routing tiebreaks above).

## Coverage clearance per block

For master-linked apps, each finding block is classified against the primary master: `covered` (hash present in the master — a refresh clears it), `diverging` (same design element, different hash — refresh would overwrite), `surplus` (element absent from the master — separate work). These per-block clearances drive the counts (`n_covered_by_master`, `n_diverging`, `n_surplus`) and the work states in the [[schema/tables/modernization-worklist]].

## Dismissed non-issues (configured false positives)

Admins can dismiss individual findings as non-issues via instance
configuration (`etl_modernization_ignore_codeblocks` — finding-grain
`code_hash|insight_id` pairs). The canonical case is **runtime-guarded vendor
code**: the stock HCL mail template's Nomad findings (OLE in the memo form,
LS2J in Mail Merge) fail only *if executed* on Nomad, and HCL guards the
entry points — an admin can legitimately declare them non-issues.

Two things hold at once, and answers must keep them straight:

- **Dismissed findings count nowhere.** Routing (`ready`), every count,
  reach, plan KPIs, worklist rows — all computed as if the finding did not
  exist. A block leaves the work surfaces only when ALL its findings are
  dismissed.
- **Dismissed findings are documented, not hidden** (since the 2026-07
  revision): they appear in the vector's `dismissed_items` array
  (`assessment: 'non_issue'`) with `n_dismissed_items` as the count, and a
  fully-dismissed app's `ready` display says so. iDNA records that a match
  WAS found and was assessed as a non-issue — auditable. Reconciliation:
  raw findings = active `items` entries + dismissed entries (see
  [[howtos/query-modernization-views]]).

Dismissals carry no who/when/why (instance configuration, not attributed
decisions) — never invent an actor or date for them.

## What the vector does NOT do (scope honesty)

- The rulesets are **code-only**. The design surface (forms, views, actions as such) is a known product gap, deliberately out of scope — never imply otherwise.
- **No effort estimates.** Delta-diffing script versions to size an edit is re-development work, not planning; the model counts blocks and containers only.
- iDNA **locates and quantifies** the work; the fixing itself happens outside iDNA, in Domino Designer / the Notes environment.

## Relationship to lifecycle

[[concepts/lifecycle-vector]] answers *whether to keep* an app; the modernization vector answers *how to get the keepers green*. The vector exists **only for `keep/%` focus apps** — but sunset apps still matter as evidence: they count in shared-block reach and appear as design-family peers. Seeing sunset apps referenced from a keep-only analysis is intentional, not a bug.

## Answer-shape rules for this topic

- **Aggregate through the template lens.** "Which apps are affected" answers group by route/session (master cohorts, standard upgrades, unify families, one-offs) — never a flat N-app finding list. A Notes developer expects help leveraging their template environment; a flat list loses credibility instantly.
- **Advisory on sequencing, firm on risks.** Recommend orderings with their rationale ("the plan ranks X first because it clears 29 apps"); never soften risk warnings.
- **Every number must reconcile.** When a user doubts a figure, offer the reconciliation path (see [[howtos/query-modernization-views]]).
