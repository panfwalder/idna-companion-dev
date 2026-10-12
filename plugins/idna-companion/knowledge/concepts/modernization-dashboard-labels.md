---
id: concepts/modernization-dashboard-labels
title: Modernization Dashboard Labels
type: concept
status: verified
tags: [domain/idna, topic/modernization, topic/dashboards, topic/64bit, topic/nomad]
related:
  - concepts/about-modernization-vector
  - concepts/about-modernization-plan
  - concepts/about-template-lineage
  - concepts/design-families
  - schema/tables/modernization-vector
  - schema/tables/modernization-plan
  - schema/tables/modernization-worklist
  - schema/tables/modernization-session-steps
  - schema/tables/modernization-lineage-dossier
  - schema/tables/template-ancestry
  - schema/tables/code-block-hcl-origin
  - howtos/find-app-lifecycle-dashboards
requires_capability: none
source: iDNA modernization dashboards (Modernization Overview, Modernization Readiness, Modernization Planning, Template Lineage, Modernization Session Work List) - texts, card label expressions and column titles read 2026-10-09, the owner's label changes verified 2026-10-12; raw values per ETL scripts 226, 284, 285, 286, 288 (iDNA_Applications_ETL); Companion plan dashboard-catalog W-4a
kbase_sources:
  - metabase-10000043-modernization-overview
  - metabase-10000045-modernization-planning
  - metabase-10000048-modernization-readiness
  - metabase-10000049-template-lineage
  - metabase-10000050-modernization-session-work-list
last_reviewed: 2026-10-12
---

# Modernization Dashboard Labels

## What it is

iDNA's modernization dashboards (Modernization Overview, Modernization Readiness, Modernization Planning, Template Lineage and Modernization Session Work List) show the modernization values under readable labels. Users quote those labels: "what does *Resolve Divergence* mean?", "why is this application *Ready (No Fix Needed)* when it has findings?". This page maps each label to the value behind it; the linked pages explain the value.

- Answer from the explaining page, and use the user's label in the answer.
- The values come from the iDNA analysis. The labels are dashboard content and can differ between iDNA versions: an older version may show the raw value (for example `review`) or another label. If a user quotes a label this page does not list, ask where they see it instead of guessing: as an `AskUserQuestion` that names the five dashboards and "elsewhere", with the column as free text (in prose only where `AskUserQuestion` is not available).

## Modernization topic

The **Modernization Topic** filter chooses the compatibility target, the ruleset (for example Notes 64-bit readiness or Nomad compatibility; `ruleset_id`, `ruleset_name` in the views). Vector, plan and work list are computed per topic, so one application may be ready for one topic and still need work for another. Other wiki pages call it the target or the ruleset.

## Vector and session labels

Modernization Readiness and Modernization Planning group applications and sessions under these labels; Planning's bar chart titles its axis "Modernization Vector / Work Session Type".

| Label | Modernization vector | Plan session kind |
|---|---|---|
| Ready (No Fix Needed) | `ready`, `ready/with_notes` | none: ready applications are in no session |
| Fix Master Template | `template/remediate_master`, with or without a qualifier | `template/remediate_master` |
| Rebase Master Template | `template/rebase_ancestor`, with or without a qualifier | `template/rebase_ancestor` |
| Upgrade Standard Template | `template/upgrade_standard`, with or without a qualifier | `template/upgrade_standard` |
| Similar to Estate Master | `template/investigate_inheritance/similar_master` | `template/investigate_inheritance` |
| Similar to Standard Template | `template/investigate_inheritance/similar_standard` | `template/investigate_inheritance` |
| Unify Similar Apps | `template/unify_similar` | `template/unify_similar` |
| Individual Code Fixes | `direct/remediate_code` | `direct/remediate_code` |

The values and their routing: [[concepts/about-modernization-vector]]; the sessions: [[concepts/about-modernization-plan]].

- **Ready (No Fix Needed) includes ready with notes.** Earlier versions of these dashboards call the group Ready (No Findings). Those applications do carry findings, but every active finding is current HCL template code: there is no fix to author, and they count as ready. For them, say "ready, with notes on current HCL template code", not "no findings". An application whose findings were all dismissed as non-issues reads `ready` and still documents them; [[concepts/about-modernization-vector]] separates active findings from documented non-issues. Visible findings alone do not settle an application's state: check its vector for the same modernization topic.
- **Needs Fix.** The Overview tab of Modernization Readiness combines every label except Ready (No Fix Needed) into Needs Fix.
- **Investigate Inheritance.** The About tab of Modernization Planning names the session kind of both similarity labels Investigate Inheritance.
- **Similar to Estate Master is broader than the `(estate master)` tag.** It covers every similarity target operated in the estate, custom-named or standard-named. The vector's display sentence tags the same target `(estate master)` (custom name) or `(standard master)` (standard name).

## Modernization Overview: Sankey labels

The Sankey of Modernization Overview reads from left to right: the root node Focus Replica Sets (Keep); the lifecycle vector (Keep: Modernize, Keep: Qualify); the track within each lifecycle branch (Keep/Modernize: Ready, Keep/Modernize: Template, Keep/Modernize: Direct, Keep/Qualify: Ready, Keep/Qualify: Template, Keep/Qualify: Direct); then the action. With the Show Details filter off (the default) the Sankey ends at the action level, with seven nodes: Ready: No Critical Issues Identified (both ready states), Template: Remediate Master, Template: Upgrade Standard, Template: Rebase Ancestor, Template: Investigate Inheritance, Template: Unify Similar and Direct: Remediate Code. With Show Details on, and in the About tab, each vector appears as `Track/Action: Detail`:

| Label | Vector |
|---|---|
| Ready/No Critical Issues Identified: Target Compatible As Is | `ready` |
| Ready/No Critical Issues Identified: Attention Notes Available | `ready/with_notes` |
| Template/Remediate Master: Fully Covered By Master | `template/remediate_master` |
| Template/Remediate Master: Unscanned Master Template | `template/remediate_master/unscanned_master_template` |
| Template/Remediate Master: Diverging Code Blocks | `template/remediate_master/diverging_code_blocks` |
| Template/Remediate Master: Surplus Code Blocks | `template/remediate_master/surplus_code_blocks` |
| Template/Upgrade Standard: Fully Covered By Master | `template/upgrade_standard` |
| Template/Upgrade Standard: Unscanned Master Template | `template/upgrade_standard/unscanned_master_template` |
| Template/Upgrade Standard: Diverging Code Blocks | `template/upgrade_standard/diverging_code_blocks` |
| Template/Upgrade Standard: Surplus Code Blocks | `template/upgrade_standard/surplus_code_blocks` |
| Template/Rebase Ancestor: Fully Covered By Master | `template/rebase_ancestor` |
| Template/Rebase Ancestor: Diverging Code Blocks | `template/rebase_ancestor/diverging_code_blocks` |
| Template/Rebase Ancestor: Surplus Code Blocks | `template/rebase_ancestor/surplus_code_blocks` |
| Template/Investigate Inheritance: Similar Master | `template/investigate_inheritance/similar_master` |
| Template/Investigate Inheritance: Similar Standard | `template/investigate_inheritance/similar_standard` |
| Template/Unify Similar: Design Family #n | `template/unify_similar` |
| Direct/Remediate Code: One Off Remediation | `direct/remediate_code` |

- **Fully Covered By Master** is the vector without a qualifier: no warning applies.
- **Design Family #n**: n is the family's `design_family_id` (see [[concepts/design-families]]).

## Work kinds

Every work item (one code block in one session) has a work kind (`work_kind`). Modernization Planning and the Modernization Session Work List label it differently:

| `work_kind` | Planning (bar chart) | Work List (Work Action) | Meaning |
|---|---|---|---|
| `author` | Author Fix | Author | the authoring home: the fix is first written in this session |
| `apply` | Apply Authored Fix | Apply | reuse a fix written in another session (see Work Authored In) |
| `review` | Resolve Divergence | Resolve Divergence | a diverging application copy: a merge decision before the master refresh, never a paste |
| `scan_first` | Scan Template First | Scan Template First | the master is unscanned, so the work is unknown until it is scanned |
| `verify_update` | Verify after Template Update | Verify Update | a check after the template update |
| `cleared_by_update` | Cleared by Template Update | Cleared by Update | removed by the template update; not hand work |
| `hcl_owned` | HCL Owned | HCL Owned (read "HCL Owned / Inherit" when the scope is HCL Owned too; the About tab calls it Inherit (HCL Owned)) | current HCL template code; not a fix |

`author`, `apply`, `review` and `scan_first` are the work; `verify_update` is a real check after the update; `cleared_by_update` and `hcl_owned` are notes, never fixes. Definitions: [[schema/tables/modernization-worklist]].

## Work scopes

The Work List's Work Action column shows the block's work scope and work kind together as "Scope / Action", for example "Cross-Session / Apply" or "Session / HCL Owned". A block that is HCL Owned in both reads "HCL Owned / Inherit".

| Work scope | `scope` | Meaning |
|---|---|---|
| Session | `session` | one authoring home, and the block lies in one session home |
| Cross-Session | `cross_session` | one authoring home, and the block spans more than one session home |
| Inherited from Standard | `inherited_standard` | no owning session; resolved by the template upgrade or rebase |
| HCL Owned | `hcl_owned` | no owning session; current HCL template code |

Scope describes the session homes; Work Action names what this session does with the block. Definitions: [[schema/tables/modernization-worklist]] (`scope`).

## Modernization Planning: Work Session Overview

**Modernization Sessions**, one row per session ([[schema/tables/modernization-plan]]):

| Column | Value |
|---|---|
| Rank | `item_rank`: applications the session alone makes ready, then applications, then usage |
| Session Title | `item_display` |
| Applications | `n_apps` |
| UAD-90d Cleared | `uad_90d_cleared`: the 90-day user access days of the applications that only this session makes ready ("usage cleared, 90 days") |
| Blocks to Fix | `n_master_blocks_to_fix`; where that is empty, `n_app_fix_items` |
| Master Unscanned | `master_unscanned` |

The number of applications the session alone makes ready (`n_apps_green`) drives the rank but is not a column.

**Affected Replica Sets**, the session's applications ([[schema/tables/modernization-vector]]):

| Column | Value |
|---|---|
| Title | the application |
| Items To Fix | `n_fix_items` |
| Shared Fixes, Solo Fixes | `n_shared_items`, `n_solo_items`: blocks shared with other applications / unique to this one |
| Covered By Master | `n_covered_by_master` |
| Diverging from Master | `n_diverging` |
| Surplus to Master | `n_surplus` |
| Declared Template(s) | the template names the application declares (`replica_set.templates_inherits_from`) |
| Ancestor Name, Ancestor Kind, Ancestor Similarity | `ancestor_name`, `ancestor_kind`, `ancestor_similarity` (rebase applications) |

A declared name is not necessarily a master operated in the estate (see "Declared is not operated" in [[concepts/about-modernization-vector]]).

**Session Steps** ([[schema/tables/modernization-session-steps]]): Step (`step_rank`), Type (`step_kind`), Description (`step_display`), Items Count (`n_items`), Item Type (`item_source`), and Drill-Down: where a step has one, it opens that step's blocks in the Work List dashboard; on the re-apply and carry-over check steps (`reapply_customizations`, `verify_carryover`) it opens the master in Template Lineage.

## Modernization Session Work List

**Work Items** ([[schema/tables/modernization-worklist]], the by-session view):

| Column | Value |
|---|---|
| Work Item | `item_display` |
| Work Action | work scope and work kind, see above |
| Has HCL Origin | the block has an HCL origin note (`hcl_origin`) |
| Work Authored In | where the fix is written: "This Session (<session kind>)" or "<anchor> (<session kind>)". Blocks without an authoring session (`authored_in` empty) show the HCL origin note when the code is current HCL template code, otherwise "No authoring: resolved by the template update"; Work Action says whether the update clears such a block or it is checked after the update. |
| Code Block Drill-Down | opens the block's code and its occurrences |
| Severity | the finding's severity |
| Reach | `reach`: kept applications carrying the block |
| Is Surplus To Master | fix, rebase and upgrade sessions with a scanned master only: true when this session's master is not recorded as a fix location for the block (its name is not in `master_targets`). The flag alone does not decide the work: follow Work Action (resolve a divergence before the refresh; author no fix for HCL Owned code). |
| Code Block Origin | the session's applications that carry the block as an apply site |
| HCL Origin | `hcl_origin`: the HCL template family and the template versions that contain this code |
| Is HCL Current, HCL In Current | `is_hcl_current`, `hcl_in_current`: the same block fact, true when the block's exact code is in the newest cataloged member of at least one live HCL template family in this iDNA build |
| Block Verdict | `block_verdict`: the lineage verdict in rebase and upgrade sessions |
| Code Diverging from Master | `has_review`: this session includes at least one diverging application occurrence of the block. The merge decision is due before the refresh even when the row's Work Action is Author or Apply. |

"Current" HCL code means iDNA's shipped reference catalog, not your installed Domino release or every newer HCL release; see [[schema/tables/code-block-hcl-origin]].

**Design/Code Elements**: Design Documents (Design Doc Type, Design Doc Name, Code Blocks) and Code Elements per Replica Set (Code Element Type, Code Element Name, Application, Finding Name, Severity, Code Block Drill-Down, Designer Link, Notes Noteid, HCL Origin, Design Document Type, Design Document Name). Designer Link opens the element in Domino Designer.

## Template Lineage

**Template Design Elements** ([[schema/tables/modernization-lineage-dossier]], element rows): Template Name, Item Type (`row_kind`), Delta Type (`delta_class`: inherited, modified or added; the default filter shows modified and added), Design Doc Name, Design Doc Type, Count Hashes (`n_hashes`), Count Custom Hashes (`n_custom_hashes`), Count Removed Ancestor Hashes (`n_removed_ancestor_hashes`).

**Template Insights Findings** (finding rows): Template Name, Finding Name, Design Doc Name, Design Doc Type, Code Element Name, Code Block Drill-Down, Compared Ancestor, Verdict, Block In Ancestor, Block In Latest.

Compared Ancestor shows how the master was compared (`classifier_mode`):

| Label | `classifier_mode` |
|---|---|
| Estate-Scanned Template | `A` |
| iDNA Reference Template | `B` |
| Standard Template | `F` |

Verdict shows the raw verdict values (`custom_fix_required`, `reapplied_with_element`, `cleared_by_rebase`, `cleared_by_upgrade`, `persists_hcl_owned`, `inherited_latest_unknown`); their meanings are in [[schema/tables/modernization-lineage-dossier]]. `reapplied_with_element` is inherited code in an element the fork added (or whose element is not identified); inherited code in a modified element follows the newest version's fate.

**Template Ancestor Overview** ([[schema/tables/template-ancestry]]): Ancestor Name, Ancestor Type (Standard Template), Ancestor Source (`ancestor_kind`: Estate-Scanned Template for `standard_estate`, iDNA Reference Template for `standard_shipped`; both empty for a master that is itself a standard template), Ancestor Similarity, Template Family, Latest Template, Latest Domino Version, Latest Source, Is Rename Only, Design Versions in Estate.

**Template Lineage Overview**: Analyzed Elements in Template, Non-Inherited Elements in Template (Element Overview tab), and the findings counts for the two built-in modernization topics (Findings for Notes 64bit Readiness, Findings for Nomad Compatibility).

## Modernization Overview: Replica Set List

The Replica Set List tab shows the vector table ([[schema/tables/modernization-vector]]), one row per kept application and topic, with these columns in this order:

| Title | Column |
|---|---|
| Replica ID, Title, Lifecycle Vector | `replicaid`, `title`, `lifecycle_vector` |
| Modernization Topic | `ruleset_name` |
| Modernization Vector, Modernization Vector Description | `modernization_vector`, `modernization_vector_display` |
| Items to Fix | `n_fix_items` |
| Items Shared Fix, Items Solo Fix | `n_shared_items`, `n_solo_items` |
| Items Covered By Master, Items Diverging, Items Surplus | `n_covered_by_master`, `n_diverging`, `n_surplus` |
| Most Similar Template, Most Similar Pct | `most_similar_template`, `most_similar_pct` |
| Design Family ID, Design Family Peers | `design_family_id`, `design_family_peers` |
| Peers Keep Modernize, Peers Keep Qualify, Peers Sunset | `peers_keep_modernize`, `peers_keep_qualify`, `peers_sunset` |
| Last Accessed | `last_accessed` |
| User Access Days (Last 365d), User Access Days (Last 90d) | `user_access_days_last365d`, `user_access_days_last90d` |
| N Dismissed Items, Dismissed Items | `n_dismissed_items`, `dismissed_items` |
| Declared Template(s) | the declared template names (`replica_set.templates_inherits_from`) |
| Ancestor Name, Ancestor Kind, Ancestor Similarity | `ancestor_name`, `ancestor_kind`, `ancestor_similarity` |
| N Cleared After Update, N Hcl Notes | `n_cleared_after_update`, `n_hcl_notes` |
| Detail Masked | `detail_masked` |
