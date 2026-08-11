---
id: concepts/nomad-web-mobile-compatibility
title: Nomad Web/Mobile Compatibility
type: concept
status: verified
tags: [domain/notes, topic/nomad, topic/modernization]
related:
  - concepts/compatibility-ruleset-routing
  - concepts/nomad-web-mobile-remediation-profile
  - concepts/nomad-web-mobile-findings-taxonomy
  - schema/tables/design-insights-summary-nomadwebmobile
  - schema/tables/design-insight-overview-byrepset
  - playbooks/source-code-remediation/remediation-routing
requires_capability: none
source: docs/reference/nomad-migration-technical-reviewed.md + docs/reference/ref_migration_nomadweb.md
last_reviewed: 2026-04-17
---

# Nomad Web/Mobile Compatibility

## What it is

Enabling HCL Notes applications for Nomad Web and Nomad Mobile can expose compatibility issues in code that relies on desktop-only features such as OLE/COM automation, local file system access, Windows shell commands, and client-specific UI operations not available in browser or mobile environments.

iDNA scans source code against a ruleset of known-incompatible patterns and produces findings per affected code element.

## iDNA ruleset

- **Ruleset ID:** `a8efac35-3e54-48f4-aa27-4c342529ed0a`
- **Ruleset name:** Nomad Web, iOS, Android Compatibility
- **Summary table:** `cqx_data.design_insights_summary_nomadwebmobile`
- **Detail table:** `cqx_data.design_insight_overview_byrepset`

## Routing binding

- `summary_table = cqx_data.design_insights_summary_nomadwebmobile`
- `ruleset_id = a8efac35-3e54-48f4-aa27-4c342529ed0a`
- shared routing rule: [[concepts/compatibility-ruleset-routing]]
- bound target profile: [[concepts/nomad-web-mobile-remediation-profile]]
- findings taxonomy: [[concepts/nomad-web-mobile-findings-taxonomy]]

## How the playbook uses this concept

Nomad compatibility can route into the shared source-code remediation playbook because the consulting pattern is largely the same: size exposure, triage the working set, identify template-based paths, identify shared-code paths, and then plan the work.

The important boundary is authority: the current detailed methodology source is still the 64-bit migration docx, so Nomad-specific guidance in this wiki should stay limited to what is actually evidenced in Nomad sources.

## Sources

CoCo seed material.
