---
id: concepts/notes-64bit-findings-taxonomy
title: Notes 64-bit Findings Taxonomy
type: concept
status: verified
tags: [domain/notes, topic/64bit, topic/modernization]
related:
  - concepts/notes-64bit-compatibility
  - concepts/notes-64bit-remediation-profile
  - schema/tables/design-insights-summary-notes64bit
  - schema/tables/design-insight-overview-byrepset
  - playbooks/source-code-remediation/remediation-routing
requires_capability: none
source: docs/reference/64bit-migration-technical-reviewed.md + docs/reference/_research/2026-04-17-compatibility/idna-finding-taxonomy.md
last_reviewed: 2026-07-23
---

# Notes 64-bit Findings Taxonomy

## Purpose

Compact classification of Notes 64-bit findings for agent retrieval and remediation routing.

## Strong incompatibility signals

- Native call and pointer-safety risk:
  `Declare` signatures returning or accepting `Long` where pointer or Windows HANDLE semantics are involved, plus `Type ... End Type` structures used with native APIs.
- Windows integration and dependency risk:
  Office-style `CreateObject`, `RegQueryValue` with 32-bit registry-view assumptions, `EmbedObject` outside simple attachment use, and native library loading patterns.
- Java/runtime modernization impact:
  references to Java packages or classes removed across Notes 12 and Notes 14 runtime changes — including the Java EE/CORBA family (`javax.xml.bind/soap/ws`, `javax.activation/jws/rmi/activity/transaction`, `org.omg.*`) covered by the dedicated `Missing Java EE/CORBA classes (Java 17 / Notes 14)` insight. Note: `com.sun.net.httpserver` still exists in Java 17 (`jdk.httpserver` module) and is deliberately NOT flagged.
- Platform-detection fragility:
  literal `'Windows/32'` comparisons (the canonical `NotesSession.Platform = "Windows/32"` bug — the check silently fails on 64-bit clients, where Platform returns `Windows/64`). A precise dedicated insight (`References platform string 'Windows/32'`) exists alongside the broader `/32`-without-`/64` heuristic.

## Heuristic review signals

- Hardcoded 32-bit path or architecture assumptions such as `/32` without `/64` and `Program Files (x86)`.
- Windows/platform-branching cues such as string literals containing `win` or `windows`, or Java code reading `os.arch` / `os.name`.
- `IsDefined(...)` misuse:
  the rule fires only on `IsDefined("WIN32")` / `IsDefined("WIN64")` — the compile-target checks that break on 64-bit. Generic cross-platform checks (`IsDefined("WINDOWS")` / `IsDefined("UNIX")`) are legitimate and are NOT flagged.

## Known blind spots

- Mixed compile/runtime defects are only partially represented by regex findings.
  `GetAllEntriesByKey` now has a dedicated low-severity insight (`GetAllEntriesByKey (mixed 32/64 compile-runtime risk)`) — a **testing cue for mixed 32/64 estates, explicitly not an incompatibility**. `Replace(str, arr1, arr2)` and `NotesUIWorkspace.Prompt` variants remain uncovered.
- Deployment-level blockers are outside normal source scanning.
  Examples include EXTMGR add-ins, ODBC DSN bitness, NotesSQL replacement, and plugin-signing or packaging issues.
- The regex findings do not prove exact pointer semantics.
  Native signatures still need manual interpretation to distinguish true pointer or HANDLE breakage from benign `Long` usage.

## Detection engine facts (2026-07 ruleset revision)

- Patterns execute as PostgreSQL regular expressions (ARE dialect) via case-insensitive matching (`~*`) — casing concerns about individual rules (e.g. `loadlibrary`) do not apply.
- An insight matches a code block only when **ALL of its topics** match (AND); patterns within a topic are OR.
- `count_findings` on the insight summaries equals the number of matched code blocks; historical numbers for multi-topic insights (notably the 2017 mail insights) were inflated by a join defect fixed in the 2026-07 revision and dropped when the fix deployed.

## Use in the workflow

Use this taxonomy after binding the Notes 64-bit profile in [[concepts/notes-64bit-remediation-profile]]. It helps decide whether a finding points mainly to source remediation, dependency inventory, mixed-runtime testing, or deployment-level follow-up.

## Sources

Reviewed 64-bit technical compendium and normalized finding taxonomy.
