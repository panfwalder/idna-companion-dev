---
id: playbooks/source-code-remediation/classify-template-paths
title: Classify Template-Based Remediation Paths
type: playbook
status: verified
tags: [domain/idna, topic/modernization, audience/consultant]
related:
  - concepts/about-modernization-vector
  - schema/tables/modernization-vector
  - playbooks/source-code-remediation/remediation-routing
  - playbooks/source-code-remediation/triage-working-set
  - playbooks/source-code-remediation/classify-shared-code-paths
  - playbooks/source-code-remediation/build-remediation-plan
  - concepts/compatibility-ruleset-routing
  - concepts/notes-64bit-remediation-profile
  - concepts/nomad-web-mobile-remediation-profile
  - concepts/remediation-category
  - concepts/about-template-inheritance
  - concepts/database-template
  - howtos/use-source-code-remediation-mcp-tools
  - schema/tables/replica-set
  - schema/tables/design-insight-overview-byrepset
  - schema/topics/template-inheritance
requires_capability: none
source: CoCo_Playbook_64bit_Migration_Consulting.docx (manual method) + modernization-vector-integration phase-2 rewrite (consume-first)
last_reviewed: 2026-07-24
---

# Classify Template-Based Remediation Paths

## When to retrieve

Use when the customer asks whether templates can solve part of the migration, or which apps get cleared by fixing/upgrading a master.

## Goal

Identify applications whose cheapest remediation path is template-based rather than application-by-application code work — and communicate the risks correctly.

## The precomputed routes (primary path)

The vector already classifies every kept focus app's template relationship per target ([[concepts/about-modernization-vector]]):

- **`template/remediate_master`** — clean estate-master link: fix the master once, refresh clears the inheriting apps. Per-block coverage is precomputed (`covered` / `diverging` / `surplus`) against the primary master.
- **`template/upgrade_standard`** — the master is a pristine, standard-named HCL template: **upgrade it to the current HCL version, don't hand-edit it** — the fixes are already made upstream.
- **`template/investigate_inheritance/similar_master|similar_standard`** — no formal link but ≥85% design-similar to a classifiable template. The display labels the SIMILARITY TARGET three ways: `(standard master)` standard-named and operated here, `(estate master)` operated under a custom name, `(standard template)` name-recognized only. This route is a **human reconcilability call**: locate the drift, then decide re-establish vs. adopt. Partial outcomes (no link re-established) are legitimate.
- **`template/unify_similar`** — similar-but-unlinked peers, the missed multiplier ([[playbooks/source-code-remediation/classify-shared-code-paths]] and [[concepts/design-families]] carry that path).

Tool routing: estate view via `modernization_plan_overview` (filter `session_kind`); per app via `modernization_for_app` (coverage counts, master names, `master_scanned`); the master session's worksheet via `modernization_session_dossier`. Existing leverage lens: `templates_by_downstream_exposure` still answers "which template families drive the most downstream exposure" at the finding level.

## Risk communication (firm, always)

- **Diverging blocks:** a template refresh **overwrites** the app's local variants — review the diverging blocks before any refresh. Never soften this.
- **Unscanned master** (`/unscanned_master_template`): coverage is UNKNOWN (NULL, not zero); the first action is a Domino-admin ask — place the master on a server iDNA scans — not a fix.
- **Surplus blocks:** the master session alone won't turn the app fully green; residual work remains (the worklist itemizes it).

## Strategy rule (unchanged)

Template linkage does not automatically mean the target problem is solved by a refresh — the finding family still matters. If the target profile says the dominant issue is unsupported runtime behavior (common for Nomad), a template relationship changes *where* the redesign happens, not whether it is a redesign.

## Ask the customer

- Are updated standard templates available for the template families in scope?
- Is the customer willing to validate and possibly re-establish broken template relationships?
- Are there known customizations that make template refresh risky? (The diverging-block review answers this with evidence.)

## Manual fallback — custom rulesets / what-if analysis

The pre-vector method classified Category A (clean template association) and Category B (broken link, strong similarity) by hand — the mapping to vector routes is on [[concepts/remediation-category]]. For rulesets outside the precompute (privileged sessions):

Intent: find affected applications with clean template associations.

```sql
SELECT rs.title, rs.templates_inherits_from, dis.count_findings,
  dis.findings_veryhigh, dis.findings_high, rs.user_access_days_last90d
FROM <summary_table> dis
JOIN cqx_data.replica_set rs USING (replicaid)
WHERE rs.is_focus_db AND dis.count_findings > 0
  AND rs.templates_inherits_from IS NOT NULL
ORDER BY dis.count_findings DESC
```

Intent: find likely broken template associations.

```sql
SELECT rs.title, rs.most_similar_template_name_newest_design,
  rs.most_similar_template_similarity_newest_design,
  dis.count_findings, rs.user_access_days_last90d
FROM <summary_table> dis
JOIN cqx_data.replica_set rs USING (replicaid)
WHERE rs.is_focus_db AND dis.count_findings > 0
  AND rs.templates_inherits_from IS NULL
  AND rs.design_is_similar_to_standard_template = true
ORDER BY rs.most_similar_template_similarity_newest_design DESC
```

## Output to next task

- master/upgrade session cohorts with their coverage risk flags
- investigate list for the human reconcilability calls
- handoff to [[playbooks/source-code-remediation/classify-shared-code-paths]] for the remaining set

## Open methodological limits from the docx

- The docx asks for clearer consulting guidance on the availability/state of HCL's 64-bit-compatible standard templates — a 64-bit-specific open question, not a generic rule.

## Sources

Consume-first: the ETL-precomputed modernization surfaces. Manual method: 64-bit migration consulting docx, Stage 3 template lens.
