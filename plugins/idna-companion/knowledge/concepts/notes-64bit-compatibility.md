---
id: concepts/notes-64bit-compatibility
title: Notes 64-bit Compatibility
type: concept
status: verified
tags: [domain/notes, topic/64bit, topic/modernization]
related:
  - concepts/compatibility-ruleset-routing
  - concepts/notes-64bit-remediation-profile
  - concepts/notes-64bit-findings-taxonomy
  - schema/tables/design-insights-summary-notes64bit
  - schema/tables/design-insight-overview-byrepset
  - playbooks/source-code-remediation/remediation-routing
requires_capability: none
source: CoCo_Playbook_64bit_Migration_Consulting.docx + CoCo Copilot server/knowledge/ref_migration_notes64bit.md
last_reviewed: 2026-04-17
---

# Notes 64-bit Compatibility

## What it is

The migration from 32-bit to 64-bit HCL Notes client can expose compatibility issues in LotusScript and Formula Language code that relies on 32-bit Windows APIs, pointer arithmetic, or platform-specific calls.

iDNA scans source code against a ruleset of known-problematic patterns and produces findings per affected code element.

## iDNA ruleset

- **Ruleset ID:** `71f16a9a-8fc6-4dfe-add8-3a43c539d353`
- **Ruleset name:** Notes 64bit Readiness
- **Summary table:** `cqx_data.design_insights_summary_notes64bit`. See [[schema/tables/design-insights-summary-notes64bit]].
- **Detail table:** `cqx_data.design_insight_overview_byrepset`. See [[schema/tables/design-insight-overview-byrepset]].

## Routing binding

- `summary_table = cqx_data.design_insights_summary_notes64bit`
- `ruleset_id = 71f16a9a-8fc6-4dfe-add8-3a43c539d353`
- shared routing rule: [[concepts/compatibility-ruleset-routing]]
- bound target profile: [[concepts/notes-64bit-remediation-profile]]
- findings taxonomy: [[concepts/notes-64bit-findings-taxonomy]]

## Findings by severity

### Very High

- code patterns that will almost certainly break or produce incorrect results in the 64-bit client
- typical examples in the docx: direct 32-bit Windows API calls with `Long` pointer parameters, explicit pointer arithmetic, and `UseDoubleAsPointer` workaround patterns

### High

- code patterns with a strong likelihood of failure
- the docx mentions 32-bit registry access calls, clipboard API usage, and OLE automation patterns depending on 32-bit COM objects

### Medium

- patterns that may cause issues depending on runtime context

### Low / Informational

- awareness items that are less likely to require code changes

## How the playbook uses this concept

Applications flagged by this ruleset enter the shared source-code remediation playbook with the 64-bit summary table and 64-bit ruleset ID bound into the workflow. See [[playbooks/source-code-remediation/remediation-routing]].

## Sources

64-bit migration consulting docx and CoCo seed material.
