---
id: concepts/nomad-web-mobile-remediation-profile
title: Nomad Web/Mobile Remediation Profile
type: concept
status: verified
tags: [domain/notes, topic/nomad, topic/modernization]
related:
  - concepts/nomad-web-mobile-compatibility
  - concepts/nomad-web-mobile-findings-taxonomy
  - concepts/compatibility-ruleset-routing
  - schema/tables/design-insights-summary-nomadwebmobile
  - schema/tables/design-insight-overview-byrepset
  - playbooks/source-code-remediation/remediation-routing
  - playbooks/source-code-remediation/assess-exposure
  - playbooks/source-code-remediation/classify-shared-code-paths
requires_capability: none
source: docs/reference/nomad-migration-technical-reviewed.md + docs/reference/ref_migration_nomadweb.md
last_reviewed: 2026-07-23
---

# Nomad Web/Mobile Remediation Profile

## Purpose

The bound target profile for running the shared source-code remediation workflow against Nomad Web/Mobile compatibility findings.

## Workflow bindings

- `summary_table = cqx_data.design_insights_summary_nomadwebmobile`
- `ruleset_id = a8efac35-3e54-48f4-aa27-4c342529ed0a`
- detail table = `cqx_data.design_insight_overview_byrepset`

## Primary finding families

- Unsupported desktop automation:
  `CreateObject`, `GetObject`, `ActivateApp`, `SendKeys`, shell execution, and similar patterns that assume a desktop runtime model.
- Unsupported native call and extension model:
  LotusScript `Declare ... Lib` patterns and `UseLSX`.
- File-system and local-environment mismatch:
  directory changes, file-attribute APIs, and local path assumptions that do not map cleanly to Nomad execution.
- Platform-scoped restrictions:
  Web-only HTTP/browser restrictions, iOS-only program-launch issues, and Android-specific UI/search differences.

## Weaker signals

- The broad `NotesHTTPRequest` flag is a review cue; the *precise* detection of the unsupported Web-only methods (`Patch`, `SetHeaderField`, `ResetHeaders`, `GetResponseHeaders`) is the dedicated insight `NotesHTTPRequest unsupported methods (Nomad Web Only)` (since the 2026-07 ruleset revision).
- `Date/Time` is a weak heuristic and can catch ordinary variable assignments.
- `Lock/Unlock` needs interpretation because the regex may mix document-locking APIs with other local-resource assumptions.

## Practical interpretation

- Nomad findings often point to environment assumptions rather than pure bitness issues.
- The common outcome is redesign, replacement, conditional execution, or feature withdrawal rather than a simple signature fix.
- Shared code-block analysis remains useful when desktop-only logic is reused broadly.
- Template-based remediation can still be high leverage when the incompatible pattern originates from shared design.

## Blind spots

- The current rules do not fully represent unsupported design/runtime models such as Java agents, XPages, and web-services design elements. (LS2J is covered since the 2026-07 ruleset revision: dedicated insight `LS2J (unsupported in Nomad)`.)
- The rules do not model the bounded COM Helper exception very well.
  Nomad Web on Windows has a constrained workaround path for some COM/OLE scenarios, but that does not create desktop parity.
- Date/time issues tied to locale, timezone, and DST are not well covered by the current regex findings.

## Caveats

- A well-maintained app legitimately carries Nomad findings for runtime-guarded desktop-only features (the stock HCL mail template does — LS2J, unsupported HTTP methods, OLE, all behind `bNotNomad`-style guards). Frame findings as non-portable code needing review, not certain breakage.
- Keep Nomad-specific claims bounded to what the Nomad sources actually support.
- Do not overread Nomad 1.0.16+ "expanded C API support" as broad support for arbitrary DLL `Declare` callouts.
- Treat COM Helper as a bounded Nomad Web workaround only, not as a general relaxation of Nomad compatibility rules.

## Use in the playbook

Resolve this profile first. For **where** the fixes happen — readiness split, session queue, per-app block work — read the precomputed modernization surfaces ([[concepts/about-modernization-vector]] / [[concepts/about-modernization-plan]]; template leverage and shared-code payoff are precomputed there, not derived by hand). Use [[concepts/nomad-web-mobile-findings-taxonomy]] to decide **what kind of work** a finding is: redesign, conditional execution, bounded Web-only workaround analysis, or further target-specific review. Then run the shared tasks in [[playbooks/source-code-remediation/remediation-routing]].

## Sources

Reviewed Nomad technical compendium and Nomad reference material.
