---
id: playbooks/source-code-remediation/remediation-routing
title: Source-Code Remediation Playbook Overview
type: playbook
status: verified
tags: [domain/idna, topic/modernization, audience/consultant]
related:
  - concepts/about-modernization-vector
  - concepts/about-modernization-plan
  - concepts/compatibility-ruleset-routing
  - concepts/notes-64bit-compatibility
  - concepts/notes-64bit-findings-taxonomy
  - concepts/notes-64bit-remediation-profile
  - concepts/remediation-category
  - concepts/nomad-web-mobile-compatibility
  - concepts/nomad-web-mobile-findings-taxonomy
  - concepts/nomad-web-mobile-remediation-profile
  - playbooks/source-code-remediation/assess-exposure
  - playbooks/source-code-remediation/triage-working-set
  - playbooks/source-code-remediation/classify-template-paths
  - playbooks/source-code-remediation/classify-shared-code-paths
  - playbooks/source-code-remediation/analyze-shared-vs-solo-blocks
  - playbooks/source-code-remediation/build-remediation-plan
  - howtos/use-source-code-remediation-mcp-tools
  - howtos/query-modernization-views
  - playbooks/application-lifecycle/lifecycle-routing
requires_capability: none
source: CoCo_Playbook_64bit_Migration_Consulting.docx (manual method) + modernization-vector-integration phase-2 rewrite (consume-first)
last_reviewed: 2026-07-24
---

# Source-Code Remediation Playbook Overview

## When to retrieve

Use this page when the agent needs to route a customer into the right part of a compatibility-oriented source-code remediation workflow, explain the overall project shape, or decide what information is missing before moving forward.

## Goal

Guide a customer from "we have compatibility findings" to a prioritized remediation plan for the applications that actually matter.

## The precomputed model (primary path)

The iDNA ETL precomputes the whole classify-and-plan pipeline per compatibility target: the **modernization vector** routes every kept focus app ([[concepts/about-modernization-vector]]), the **plan** ranks the work sessions, and the **worklist** itemizes every code fix at block grain ([[concepts/about-modernization-plan]]). The consultant's job therefore starts at *interpreting the plan*, not at deriving it. Findings only decide ready vs. not-ready; everything after that is template relationships — which is why the answers must aggregate through the template lens, never as flat finding lists.

## Routing map (primary)

- "What breaks on 64-bit?" / "Which apps aren't Nomad-ready?" / "How ready are we?" → readiness split + KPI: `modernization_plan_overview` (or the vector view); background in [[playbooks/source-code-remediation/assess-exposure]] for the exposure-vs-readiness distinction
- "How big is the finding volume / how severe?" → `exposure_summary` / `most_affected_apps` — the volume lens ([[playbooks/source-code-remediation/assess-exposure]])
- "Where do we start? What pays off most?" → `modernization_plan_overview` — the session queue ranks by apps cleared
- "Is app X ready / what fixes X?" → `modernization_for_app`
- "What exactly do I do in session X?" → `modernization_session_dossier` ([[playbooks/source-code-remediation/build-remediation-plan]])
- "Which fixes do we author once and reapply?" → [[playbooks/source-code-remediation/analyze-shared-vs-solo-blocks]]
- "Can standard templates solve this?" → [[playbooks/source-code-remediation/classify-template-paths]]
- "Which shared code fixes give the best payoff?" → [[playbooks/source-code-remediation/classify-shared-code-paths]]
- "Which affected apps do we really need to fix?" → [[playbooks/source-code-remediation/triage-working-set]]

## The consulting narrative (three tiers)

Narrate the plan multipliers-first — the computed rankings stay as-is (sessions by apps cleared, blocks by reach); the tiers are the presentation:

1. **Custom-master multipliers** (`remediate_master`): fix the master once, a refresh clears the cohort. Broken/lost template relations (`unify_similar`, `investigate_inheritance`) belong here as the secondary opportunity.
2. **Standard-template multipliers** (`upgrade_standard`, `rebase_ancestor`): port to the current HCL version — the fixes are already made and HCL maintains them. Detached forks of a standard are rebased onto the current version of their lineage, then their customizations are re-applied ([[concepts/about-template-lineage]]).
3. **Piece-work from the worklist**, heavy-impact first: shared script libraries down to unique solo blocks.

The work itself happens **outside iDNA**, in Domino Designer, by the customer's developer or admin — iDNA locates and quantifies it. Stay advisory on sequencing; keep risk statements firm (a refresh overwrites diverging blocks — review first).

## Remediation strategy rule (unchanged)

Do not treat all findings as generic "code fixes." After target routing, classify the dominant finding type into: source remediation, redesign, conditional execution, dependency inventory, or deployment follow-up — the target profiles and taxonomy pages say which path is normal for the current ruleset. The vector routes tell you *where* to fix; the taxonomy tells you *what kind of work* the fix is.

## Required foundations

- [[concepts/about-modernization-vector]] and [[concepts/about-modernization-plan]]
- [[concepts/compatibility-ruleset-routing]] — bind the target before anything else
- the target profiles and findings taxonomies (64-bit and Nomad)

## Manual fallback — custom rulesets / what-if analysis

The original consulting method — a narrowing funnel of quantify exposure → triage → classify by cheapest path (the A/B/C/D scheme, see [[concepts/remediation-category]]) → sequence — remains valid when the target ruleset is NOT in the modernization precompute, or for ad-hoc what-if analysis. The manual steps live in the fallback sections of the task pages ([[playbooks/source-code-remediation/assess-exposure]], [[playbooks/source-code-remediation/triage-working-set]], [[playbooks/source-code-remediation/classify-template-paths]], [[playbooks/source-code-remediation/classify-shared-code-paths]], [[playbooks/source-code-remediation/build-remediation-plan]]). The core discipline holds either way: do not jump straight from findings to code fixes.

## Output

A routed next step into one of the pages/tools above, not a broad narrative answer.

## Sources

Consume-first: the ETL-precomputed modernization surfaces. Manual method: 64-bit migration consulting docx.
