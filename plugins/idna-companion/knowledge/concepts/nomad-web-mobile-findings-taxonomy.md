---
id: concepts/nomad-web-mobile-findings-taxonomy
title: Nomad Web/Mobile Findings Taxonomy
type: concept
status: verified
tags: [domain/notes, topic/nomad, topic/modernization]
related:
  - concepts/nomad-web-mobile-compatibility
  - concepts/nomad-web-mobile-remediation-profile
  - schema/tables/design-insights-summary-nomadwebmobile
  - schema/tables/design-insight-overview-byrepset
  - playbooks/source-code-remediation/remediation-routing
requires_capability: none
source: docs/reference/nomad-migration-technical-reviewed.md + docs/reference/_research/2026-04-17-compatibility/idna-finding-taxonomy.md
last_reviewed: 2026-07-23
---

# Nomad Web/Mobile Findings Taxonomy

## Purpose

Compact classification of Nomad Web, iOS, and Android findings for agent retrieval and remediation routing.

## Strong incompatibility signals

- Unsupported desktop automation:
  `CreateObject`, `GetObject`, `ActivateApp`/`AppActivate`, `SendKeys`, and shell execution (`Shell(...)` in call and assignment form) usually indicate a desktop runtime model that Nomad does not provide.
- Unsupported native call and extension model:
  LotusScript `Declare ... Lib` patterns and `UseLSX` indicate native or extension-based behavior that is generally not portable into Nomad.
- LS2J:
  LotusScript-to-Java bridging is unsupported in Nomad and has a dedicated insight (`LS2J (unsupported in Nomad)`). Common estate carriers include mail-template features (Mail Merge, DOMI OAuth).
- File-system and local-environment mismatch:
  `ChDir`/`ChDrive`, `GetFileAttr`, and `SetFileAttr` indicate assumptions about local filesystem behavior that do not map cleanly to Nomad execution, especially on Web (Java `ftp.chdir(...)` method calls are excluded — not the LotusScript statement).
- Platform-specific restrictions:
  `OSLoadProgram` is iOS-specific, `Edit/Find` is Android-specific, and HTTP behavior on Nomad Web has browser-specific limits.
- Unsupported `NotesHTTPRequest` methods on Nomad Web:
  a precise insight (`NotesHTTPRequest unsupported methods (Nomad Web Only)`) detects the documented Web-only gaps `Patch`, `SetHeaderField`, `ResetHeaders`, and `GetResponseHeaders` specifically.

## Heuristic review signals

- The broad `NotesHTTPRequest (Nomad Web Only)` flag remains a review cue (it also catches `CreateHTTPRequest()`); the *precise* unsupported-method detection is the dedicated insight listed under strong signals. Browser-level restrictions such as CORS still require manual review.
- `Date/Time` is a weak heuristic.
  It no longer misfires on `==` comparisons or cross-line accidents, but it still does not directly detect the more meaningful Nomad timezone or DST-sensitive risks.
- `Lock/Unlock` needs careful interpretation.
  The rule may mix document-locking APIs with other client-state or local-resource assumptions.

## Known blind spots

- The current rules do not fully represent unsupported design/runtime models such as Java agents, XPages, and web-services design elements. (LS2J is no longer a blind spot — it has a dedicated insight, see strong signals.)
- The findings do not model the bounded COM Helper exception well.
  Nomad Web on Windows has a constrained workaround path for some COM/OLE scenarios, but that does not create desktop parity.
- The current ruleset is still too coarse across target families.
  Web, iOS, and Android often need separate interpretation.
- Date/time compatibility risks tied to locale, timezone, and DST are not well covered by the current regex findings.

## Calibration: findings measure non-portable code, not breakage

The stock HCL mail template itself carries a body of genuine Nomad findings (LS2J in Mail Merge and DOMI OAuth, unsupported `NotesHTTPRequest` methods, OLE in the memo form) — features that fail deterministically *if executed* on Nomad, but whose entry points HCL guards at runtime (visible `bNotNomad` checks). A well-maintained app legitimately carries Nomad findings for guarded desktop-only features. Treat vendor templates as a calibration baseline, not as "dirty"; frame findings as the presence of non-portable code needing review, never as certain breakage.

## Detection engine facts (2026-07 ruleset revision)

- Patterns execute as PostgreSQL regular expressions (ARE dialect) via case-insensitive matching (`~*`).
- An insight matches a code block only when **ALL of its topics** match (AND); patterns within a topic are OR.
- `count_findings` on the insight summaries equals the number of matched code blocks; historical numbers for multi-topic insights were inflated by a join defect fixed in the 2026-07 revision and dropped when the fix deployed.

## Use in the workflow

Use this taxonomy after binding the Nomad profile in [[concepts/nomad-web-mobile-remediation-profile]]. It helps decide whether a finding points to redesign, conditional execution, bounded Web-only workaround analysis, or further target-specific review.

## Sources

Reviewed Nomad technical compendium and normalized finding taxonomy.
