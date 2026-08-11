---
id: concepts/notes-64bit-remediation-profile
title: Notes 64-bit Remediation Profile
type: concept
status: verified
tags: [domain/notes, topic/64bit, topic/modernization]
related:
  - concepts/notes-64bit-compatibility
  - concepts/notes-64bit-findings-taxonomy
  - concepts/compatibility-ruleset-routing
  - schema/tables/design-insights-summary-notes64bit
  - schema/tables/design-insight-overview-byrepset
  - playbooks/source-code-remediation/remediation-routing
  - playbooks/source-code-remediation/assess-exposure
  - playbooks/source-code-remediation/classify-shared-code-paths
requires_capability: none
source: docs/reference/64bit-migration-technical-reviewed.md + CoCo_Playbook_64bit_Migration_Consulting.docx
last_reviewed: 2026-07-23
---

# Notes 64-bit Remediation Profile

## Purpose

The bound target profile for running the shared source-code remediation workflow against Notes 64-bit compatibility findings.

## Workflow bindings

- `summary_table = cqx_data.design_insights_summary_notes64bit`
- `ruleset_id = 71f16a9a-8fc6-4dfe-add8-3a43c539d353`
- detail table = `cqx_data.design_insight_overview_byrepset`

## Primary finding families

- Native call and pointer-safety risk:
  `Declare` signatures, pointer returns, Windows HANDLE handling, and `Type` blocks used with native APIs.
- Mixed compile/runtime defects:
  code that can compile on one bitness and fail or misbehave on the other even when the source pattern itself looks reasonable.
- Windows integration and dependency risk:
  Office-style `CreateObject`, registry-view assumptions, embedded-object behavior, and dynamic library loading.
- Java/runtime modernization:
  removed Java packages, Notes 14+ runtime changes, and packaging or plugin implications.

## Weaker signals

- Hardcoded 32-bit path and architecture assumptions such as `/32` without `/64` or `Program Files (x86)`.
- Broad Windows/platform cues such as `win`, `windows`, `os.arch`, and `os.name`.
- `IsDefined(...)` misuse:
  the rule fires only on `IsDefined("WIN32")` / `IsDefined("WIN64")`; generic cross-platform checks (`"WINDOWS"`/`"UNIX"`) are legitimate and not flagged (since the 2026-07 ruleset revision).
- Literal `'Windows/32'` platform comparisons have their own precise insight (`References platform string 'Windows/32'`) — the canonical `NotesSession.Platform` check that silently fails on 64-bit clients.

## Practical interpretation

- This target often leads to code repair, dual-pathing, dependency replacement, or deployment cleanup rather than full redesign.
- Severity is useful for triage, but it is not a precise estimate of remediation effort.
- Shared code-block analysis is especially valuable when the same native-call or dependency pattern is reused across many applications.
- Template-based remediation remains the first high-leverage path when findings originate from inherited standard design.

## Blind spots

- Mixed compile/runtime defects are only partially covered by regex findings.
  `GetAllEntriesByKey` now has a dedicated low-severity insight — a testing cue for mixed 32/64 estates, explicitly not an incompatibility. `Replace(str, arr1, arr2)` and `NotesUIWorkspace.Prompt` variants remain uncovered.
- Deployment-level blockers sit outside normal source scanning.
  Examples include EXTMGR add-ins, ODBC DSN bitness, NotesSQL replacement, and plugin-signing or packaging problems.

## Caveats

- The docx is authoritative for the 64-bit method, but it still leaves thresholds, effort ranges, and some template-version specifics open.
- Do not infer exact remediation effort from severity alone.
- Not every `Long` in a native signature is wrong.
  The real question is whether the value is acting as a pointer or Windows HANDLE.

## Use in the playbook

Resolve this profile first. For **where** the fixes happen — readiness split, session queue, per-app block work — read the precomputed modernization surfaces ([[concepts/about-modernization-vector]] / [[concepts/about-modernization-plan]]; template leverage and shared-code payoff are precomputed there, not derived by hand). Use [[concepts/notes-64bit-findings-taxonomy]] to decide **what kind of work** a finding is: source remediation, mixed-runtime testing, dependency inventory, or deployment-level follow-up. Then run the shared tasks in [[playbooks/source-code-remediation/remediation-routing]].

## Sources

Reviewed 64-bit technical compendium and 64-bit migration consulting docx.
