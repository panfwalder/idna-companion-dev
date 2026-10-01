---
id: playbooks/source-code-remediation/classify-shared-code-paths
title: Classify Shared-Code and Unique Remediation Paths
type: playbook
status: verified
tags: [domain/idna, topic/modernization, audience/consultant]
related:
  - concepts/about-modernization-plan
  - schema/tables/modernization-worklist
  - playbooks/source-code-remediation/analyze-shared-vs-solo-blocks
  - playbooks/source-code-remediation/remediation-routing
  - playbooks/source-code-remediation/triage-working-set
  - playbooks/source-code-remediation/classify-template-paths
  - playbooks/source-code-remediation/build-remediation-plan
  - concepts/compatibility-ruleset-routing
  - concepts/notes-64bit-findings-taxonomy
  - concepts/notes-64bit-remediation-profile
  - concepts/nomad-web-mobile-findings-taxonomy
  - concepts/nomad-web-mobile-remediation-profile
  - concepts/remediation-category
  - concepts/code-block
  - howtos/use-source-code-remediation-mcp-tools
  - schema/tables/design-insight-overview-byrepset
  - schema/tables/code-source
  - schema/tables/replica-set
requires_capability: none
source: CoCo_Playbook_64bit_Migration_Consulting.docx (manual method) + modernization-vector-integration phase-2 rewrite (consume-first); hcl_owned scope and the n_hcl_current identity term (ETL ADR 0006) 2026-10-01
last_reviewed: 2026-10-01
---

# Classify Shared-Code and Unique Remediation Paths

## When to retrieve

Use when the customer asks where the biggest custom-code efficiency gains are, or which fixes are one-offs.

## Goal

Identify which remaining work can be addressed through high-impact fix-once/reapply blocks and which is isolated application-by-application remediation.

## The precomputed worklist (primary path)

`cqx_data.modernization_worklist` IS this classification, per target, at block grain ([[schema/tables/modernization-worklist]]): every code fix, ranked **reach-first**, with `scope` (`cross_session` = author once, apply in several sittings; `session` = all occurrences inside one home; `inherited_standard` = no sitting owns it, template upgrades/rebases resolve it; `hcl_owned` = no sitting owns it, the code is current HCL template code, no code fix to author — and scope is NOT the reach split), the authoring home pair (`authored_session_kind`, `authored_in`), and per-block accounting (`reach = n_cleared_by_refresh + n_persists_after_update + n_hcl_current + apply + review + scan_first`, ETL ADR 0003 as amended by ETL ADR 0006).

- Estate view: `modernization_plan_overview` (top blocks) or the worklist directly.
- Per app: `modernization_for_app` (block list with work states).
- Full analysis workflow — reach interpretation, drift, decision points, reconciliation: **[[playbooks/source-code-remediation/analyze-shared-vs-solo-blocks]]** (the dedicated playbook; this page classifies, that page analyzes).
- Block drill-down pair unchanged: `get_code_block` (source) + `find_code_block_usage` (where it appears — pass the target for per-ruleset reach).

The old vocabulary maps directly: Category C (shared blocks) ≈ worklist blocks with reach > 1; Category D (unique code) ≈ solo blocks / `direct/remediate_code` apps ([[concepts/remediation-category]] carries the full mapping).

## Strategy rule (unchanged)

Shared code only helps if the remediation strategy itself is reusable. Notes 64-bit findings often fit well (one pointer-safe or dependency-aware fix clears many applications). Nomad findings split: shared redesign candidates vs. shared desktop-only behavior needing target-specific business decisions before remediation. A widely reused block is not automatically a quick win — if the finding family implies redesign, the shared block still carries solution-design effort. And some shared code traces back to templates: the worklist's `n_masters_containing` and `n_cleared_by_refresh` say when a block really belongs to the template path.

## Ask the customer

- Prioritize the highest-impact shared fixes first, even across many applications?
- Any critical applications to use as anchors when interpreting shared-code opportunities?

## Manual fallback — custom rulesets / what-if analysis

For rulesets outside the precompute (privileged sessions):

Intent: rank shared code blocks by cross-application impact.

```sql
SELECT f.finding_name, f.severity, f.code_hash_id, f.source_code_hash,
  COUNT(DISTINCT f.replicaid) AS affected_apps,
  f.code_element_type
FROM cqx_data.design_insight_overview_byrepset f
JOIN cqx_data.replica_set rs USING (replicaid)
WHERE rs.is_focus_db
  AND f.ruleset_id = '<ruleset_id>'
GROUP BY f.finding_name, f.severity, f.code_hash_id, f.source_code_hash, f.code_element_type
HAVING COUNT(DISTINCT f.replicaid) > 1
ORDER BY COUNT(DISTINCT f.replicaid) DESC
LIMIT 20
```

Intent: show which applications contain a specific code block.

```sql
SELECT rs.title, f.finding_name, f.severity, f.code_element_type,
  f.design_document_name, rs.user_access_days_last90d
FROM cqx_data.design_insight_overview_byrepset f
JOIN cqx_data.replica_set rs USING (replicaid)
WHERE rs.is_focus_db AND f.code_hash_id = <code_hash_id>
ORDER BY rs.user_access_days_last90d DESC NULLS LAST
```

Intent: retrieve the normalized source for a code block.

```sql
SELECT code_text
FROM cqx_data.code_source
WHERE hash_id = <code_hash_id>
```

## Output to next task

- ranked fix-once work list (worklist order) + one-off residual set
- handoff to [[playbooks/source-code-remediation/build-remediation-plan]]

## Open methodological limits from the docx

- The docx mentions AI-assisted code analysis as optional/future capability, not a primary delivery assumption.

## Sources

Consume-first: the ETL-precomputed modernization worklist. Manual method: 64-bit migration consulting docx, Stage 3 code-block lens.
