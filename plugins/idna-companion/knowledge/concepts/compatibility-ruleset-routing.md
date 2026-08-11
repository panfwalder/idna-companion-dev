---
id: concepts/compatibility-ruleset-routing
title: Compatibility Ruleset Routing
type: concept
status: verified
tags: [domain/idna, topic/modernization]
related:
  - concepts/about-modernization-vector
  - schema/tables/modernization-vector
  - concepts/notes-64bit-compatibility
  - concepts/notes-64bit-remediation-profile
  - concepts/nomad-web-mobile-compatibility
  - concepts/nomad-web-mobile-remediation-profile
  - schema/tables/design-insights-summary-notes64bit
  - schema/tables/design-insights-summary-nomadwebmobile
  - schema/tables/design-insight-overview-byrepset
  - playbooks/source-code-remediation/remediation-routing
  - playbooks/source-code-remediation/assess-exposure
  - playbooks/source-code-remediation/classify-shared-code-paths
requires_capability: none
source: CoCo_Playbook_64bit_Migration_Consulting.docx + CoCo Copilot server/knowledge/ref_migration_nomadweb.md + docs/reference/cqx_data_schema_enriched_all_tables.md
last_reviewed: 2026-07-23
---

# Compatibility Ruleset Routing

## What it is

The routing rule for compatibility-oriented source-code remediation work: choose the target ruleset first, then run the shared remediation workflow with the matching summary table and ruleset identifier.

## Core rule

The remediation method is shared, but the input objects change by target. Resolve the target profile first:

| Target | Target profile | Summary table | `ruleset_id` for detail queries |
|---|---|---|
| Notes 64-bit | [[concepts/notes-64bit-remediation-profile]] | `cqx_data.design_insights_summary_notes64bit` | `71f16a9a-8fc6-4dfe-add8-3a43c539d353` |
| Nomad Web/Mobile | [[concepts/nomad-web-mobile-remediation-profile]] | `cqx_data.design_insights_summary_nomadwebmobile` | `a8efac35-3e54-48f4-aa27-4c342529ed0a` |

The shared detail table is always [[schema/tables/design-insight-overview-byrepset]].

## The vector: routing is precomputed per target

Since 2026-07 the ETL applies this routing per (kept focus app × ruleset) and publishes the result as the **modernization vector** ([[concepts/about-modernization-vector]]): readiness, fix route, session assignment, and per-block work. For "is app X ready for target Y", "which apps aren't ready", and "what do we fix first" questions, **read the precomputed vector/plan/worklist surfaces first** — the manual classification method remains for custom rulesets outside the modernization configuration and for what-if analysis. The `ruleset_id` bindings in the table above are the same ids used by `cqx_data.modernization_vector` / `modernization_plan` / `modernization_worklist`.

## How the skill should use this

1. Determine which compatibility target the customer is asking about.
2. Bind `<summary_table>` and `<ruleset_id>` from that profile.
3. Reuse the shared playbook tasks with those bindings.
4. Keep target-specific interpretation in the target profile and target concept pages, not in the shared playbook.

## Caveat

The current detailed method source is the 64-bit migration consulting docx. That is strong enough to ground the shared workflow, but not strong enough to invent Nomad-specific claims beyond what Nomad sources actually support.

## Sources

64-bit consulting docx, Nomad CoCo seed, and schema reference.
