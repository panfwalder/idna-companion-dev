---
id: playbooks/source-code-remediation/build-remediation-plan
title: Build the Source-Code Remediation Plan
type: playbook
status: verified
tags: [domain/idna, topic/modernization, audience/consultant]
related:
  - concepts/about-modernization-plan
  - schema/tables/modernization-plan
  - schema/tables/modernization-worklist
  - playbooks/source-code-remediation/remediation-routing
  - playbooks/source-code-remediation/classify-template-paths
  - playbooks/source-code-remediation/classify-shared-code-paths
  - playbooks/source-code-remediation/analyze-shared-vs-solo-blocks
  - concepts/compatibility-ruleset-routing
  - concepts/notes-64bit-findings-taxonomy
  - concepts/notes-64bit-remediation-profile
  - concepts/nomad-web-mobile-findings-taxonomy
  - concepts/nomad-web-mobile-remediation-profile
  - concepts/remediation-category
  - concepts/user-access-days
  - concepts/remediation-classification
  - howtos/use-source-code-remediation-mcp-tools
  - schema/tables/replica-set
  - schema/tables/design-insights-summary-notes64bit
  - schema/tables/design-insights-summary-nomadwebmobile
  - schema/tables/dbviews-used
  - concepts/database-complexity
requires_capability: none
source: CoCo_Playbook_64bit_Migration_Consulting.docx (manual method) + modernization-vector-integration phase-2 rewrite (consume-first)
last_reviewed: 2026-07-24
---

# Build the Source-Code Remediation Plan

## When to retrieve

Use when the customer asks how to sequence the work, what to prioritize first, or what the project plan should look like.

## Goal

Turn the precomputed work queue into a customer-ready roadmap that balances efficiency and business impact.

## The plan is precomputed (primary path)

The ordered queue exists: `modernization_plan_overview` returns the readiness KPI plus the session queue ranked by **apps cleared** (`n_apps_green` — the session KPI; usage-cleared companions break ties), and the worklist ranks every block fix reach-first. A session is one human sitting on one design container. Session detail: `modernization_session_dossier` — the worksheet with author/apply/review/scan_first rows, executed by the customer's developer in Domino Designer (iDNA locates and quantifies; it does not apply fixes).

**Narrate in three tiers** (presentation order; the computed rankings stay as-is):

1. **Custom-master multipliers** (`remediate_master`) — fix the master once, refresh clears the cohort; broken/lost relations (`unify_similar`, `investigate_inheritance`) mentioned here as the secondary opportunity.
2. **Standard-template multipliers** (`upgrade_standard`) — port to the current HCL version, don't hand-edit.
3. **Worklist piece-work**, heavy-impact first — shared script libraries down to unique solo blocks ([[playbooks/source-code-remediation/analyze-shared-vs-solo-blocks]]).

Advisory framing throughout: the ranking has a stated rationale (apps cleared); sequencing decisions — staffing, politics, deadlines — stay with the customer. Unify/investigate session counts assume human decisions resolve; partial outcomes (two masters instead of one; "not reconcilable") are legitimate exits. Risk statements stay firm: a refresh on a session with diverging blocks needs the review-first warning every time.

## The business overlay (the consultant's own contribution)

The precompute does not know business criticality. Sequence by the queue, then adjust for:

- customer-named business-critical applications (regardless of modest usage numbers)
- fixed migration deadlines (interim mitigations may beat ideal sequencing)
- dependencies between applications that must stay in sync

Within any tier, the strategy class per finding family stays explicit (code repair / redesign / conditional execution / dependency follow-up / deployment follow-up) — the target profile and taxonomy decide, not the route.

## Refinement: unused-element findings (drill-down)

Not every finding is worth remediating. A finding on a **view that is never used** is a candidate to deprioritize or exclude:

- Scope per instance from `cqx_data.database.database_views`: `count_code - count_code_used` = coded views with no built index.
- Confirm specific views with [[schema/tables/dbviews-used]], aggregated across the replica set (unused only if unused on EVERY instance).
- **This is a manual drill-down, not a tool default.** Neither the findings tools nor the modernization precompute account for view usage — apply this refinement on top of their output.
- Forms are not yet covered by this signal; views are the case available today.

## Effort guidance (unchanged — still a consulting input)

Effort ranges need consulting calibration; provide the framework, ask for assumptions, never invent confident hour estimates. Break effort into analysis / remediation / validation, and by strategy class (code repair estimable from sharedness and depth; redesign carries solution-design uncertainty; dependency and deployment follow-up involve external verification). The plan counts blocks and containers — deliberately not effort.

## Deliverable shape

- phased roadmap: the three tiers with the session queue inside each
- critical-path applications (business overlay)
- deferred/archive list from triage (with sunset-funnel dispositions)
- stated assumptions and unknowns
- recommendation to re-scan after fixes

## Manual fallback — custom rulesets / what-if analysis

For rulesets outside the precompute, the manual category-ordered method applies (A/B/C/D — mapping on [[concepts/remediation-category]]), with the ranking input query (privileged sessions):

```sql
SELECT rs.title, dis.findings_veryhigh, dis.findings_high, dis.findings_medium,
  dis.count_findings, rs.user_access_days_last90d, rs.usage_last90d,
  rs.templates_inherits_from, rs.design_is_similar_to_standard_template
FROM <summary_table> dis
JOIN cqx_data.replica_set rs USING (replicaid)
WHERE rs.is_focus_db AND dis.count_findings > 0
  AND rs.remediation_classification != 'Consolidation Potential'
ORDER BY dis.findings_veryhigh DESC, rs.user_access_days_last90d DESC NULLS LAST
```

## What to tell the customer

- This is a draft project structure, not a guaranteed estimate.
- Apps cleared per sitting is the ranking; business criticality changes the order, severity alone does not.
- Not all high findings belong in the same workstream — some are code fixes, some redesign, some deployment blockers.

## Output

A customer-ready draft remediation roadmap.

## Open methodological limits from the docx

- effort ranges are still an open consulting input
- business criticality must come from the customer, not from iDNA alone

## Sources

Consume-first: the ETL-precomputed modernization plan/worklist. Manual method: 64-bit migration consulting docx, Stage 4.
