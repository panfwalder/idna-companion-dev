---
id: concepts/about-template-lineage
title: Template Lineage (Ancestry, Rebase, Update Fate, Session Steps)
type: concept
status: verified
tags: [domain/idna, topic/modernization, topic/64bit, topic/nomad]
related:
  - schema/tables/template-ancestry
  - schema/tables/modernization-lineage-dossier
  - schema/tables/modernization-session-steps
  - concepts/about-modernization-vector
  - concepts/about-modernization-plan
  - concepts/about-template-inheritance
  - concepts/design-similarity
  - schema/tables/modernization-worklist
requires_capability: none
source: iDNA_Applications_ETL docs/decisions ADR 0002 (ancestry + rebase route), ADR 0003 (verdict-aware worklist, ready/with_notes), ADR 0004 (session steps); ETL scripts 225/226/285/286/288/289
last_reviewed: 2026-09-28
---

# Template Lineage (Ancestry, Rebase, Update Fate, Session Steps)

## The real-world pattern

Notes estates grew for decades by copying. Someone copies a standard template they like (a copy severs inheritance), changes a few things, and the result becomes "our own" master — a **detached fork** of an ancestor, often an old HCL standard template. The people who inherit such a master usually no longer know what was changed, which is exactly why they are afraid to touch it. Detached forks of standard templates are expected to be a high-volume case at customers.

iDNA recovers the lost lineage from design similarity and turns it into work that respects it (ETL ADRs 0002–0004).

## Ancestry

[[schema/tables/template-ancestry]] names, per estate master, the most similar STANDARD identity its design links to (most similar wins; an estate-operated standard breaks ties), the similarity, and the newest member of that template family — the upgrade reference. The family is a catalog fact, not a similarity fact: "upgrade to the current version of the standard template" is always the recommended target, whichever concrete version the customer picks.

## Three repair options for a master

| Master | Route | The work |
|---|---|---|
| Carries a standard name (a pristine HCL template) | `template/upgrade_standard` | Upgrade it to the current HCL version — never hand-edit standard code. |
| Own name, design derived from a standard ancestor | `template/rebase_ancestor` | Replace its design with the current version of the ancestor lineage, then re-apply the customizations. |
| Own name, no standard ancestry | `template/remediate_master` | Fix the master's code once; a refresh clears the inheriting apps. |

`rebase_ancestor` REPLACES `remediate_master` for masters with standard ancestry: editing old standard code is wasted work when the current HCL version already contains the fixes. A rename-only fork (100% similar — identical code base under another name) reads "disconnected from master …, identical code base — upgrade directly".

## The two questions the lineage dossier answers

[[schema/tables/modernization-lineage-dossier]]:

1. **"What did we change back then?"** — element deltas against the ancestor: `inherited`, `modified`, `added`.
2. **"What survives the update?"** — a verdict per flagged block: cleared by the rebase/upgrade, custom code to fix (`custom_fix_required`), carried back by a re-applied element or of unknown fate (verify after the update), or still present in the current HCL version (`persists_hcl_owned`).

## Product stance: three app states

iDNA helps customers get THEIR work done; it is not a tool for identifying bugs in HCL code (ETL ADR 0003). The customer surface has exactly three app states:

- **`ready`** — no active findings for the target.
- **`ready/with_notes`** — every active fix item is `persists_hcl_owned`: HCL-shipped code present in the current template version. The app is READY; the findings stay visible as notes, maintained by HCL. Display: "Ready - N note(s): findings in HCL-shipped code (current template version), maintained by HCL". Never presented as a session or as work — it never enters the plan or the worklist. Generic by design: no per-cause variants, no confidence percentages.
- **a work route** — a session that makes the app green.

The verdict taxonomy is a consultant drill-down, not something to overload customers with. Findings in untouched standard code are never presented as fixes to author.

## Session steps: the procedure

A template-lineage session is mostly template operations, and only its code-block middle has worklist rows. [[schema/tables/modernization-session-steps]] gives every session its ordered procedure. For a rebase that is: scan (if needed) → back up the master → replace the design → re-apply ALL customizations from the backup (including finding-free modified elements) → author fixes for remaining custom blocks → verify the fork-only elements survived → confirm cleared findings are gone → review HCL-owned notes → point member apps at the master and refresh → fix app copies the master does not cover → re-scan.

Two rules to keep in every answer:

- **Master-side work before the refresh, app-side work after it.** A refresh overwrites diverging app code. When the dossier has review rows (apps whose code diverges from the master), DECIDE the merge before the refresh — keep the variant by moving it into the master, or accept the master's version. The app-copy fix itself comes after the refresh (`fix_app_copies`). Without review rows there is no merge to decide; do not raise it. When the dossier's rows were cut, follow the tool's review-timing note — it covers the case where the review rows are among the cut ones.
- **Customizations are restored, not just flagged blocks** — a design replace can destroy fork-only elements; the verify step exists because of that.

Steps are guidance only: no completion state, never "edit the standard code".

## How to use this in answers

- **"What do I do in this session?" for a rebase or upgrade session:** lead with the procedure (session steps), then the block work beneath each step.
  - Any session: `modernization_session_dossier` returns the steps (`steps`) and the work rows in procedure order, each row naming its step.
  - If the dossier returns no `steps` field (an older MCP build): privileged sessions query `cqx_data.modernization_session_steps` directly; restricted sessions present the procedure shape from this page, together with the dossier's work rows.
- **"What did we customize?" / "What survives the update?" for one master:** `modernization_lineage_overview` gives the counts (by master name, or by an app on a rebase / upgrade / with-notes route); drill into one count with `modernization_lineage_dossier`, using exactly the values the overview shows. Any profile.
  - If these tools are absent (an older MCP build): privileged sessions query the two lineage views; restricted sessions answer from this page and say that lineage detail is not available in this session.
- **Never** tell a customer to edit a standard template's code; never present `hcl_owned` or `cleared_by_update` rows as work.
- Keep the mode-B honesty. Against a shipped reference only, say "all-custom code elements (possibly new or fully rewritten)", not "fork-only elements".
- The fixing happens outside iDNA, by the customer's developer in Domino Designer; iDNA locates and quantifies the work.
