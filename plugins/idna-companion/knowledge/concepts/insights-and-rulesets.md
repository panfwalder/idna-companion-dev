---
id: concepts/insights-and-rulesets
title: Insights and rulesets - how iDNA flags code patterns
type: concept
status: verified
tags: [domain/idna, topic/modernization]
related:
  - concepts/shipped-insight-catalog
  - howtos/find-code-patterns
  - concepts/compatibility-ruleset-routing
  - schema/tables/design-insight
  - schema/tables/design-insight-ruleset
  - schema/tables/design-insight-ruleset-member
  - schema/tables/design-insight-overview-byrepset
  - schema/topics/insights-overview-exclude
  - howtos/find-design-dashboards
requires_capability: none
source: ETL repo dda_ci/010-015 and 080-085 (catalog structure, shipped content at 1c5d046), dda_dwh/scheduled/061 (matching), dda_dwh/080 (severity scale); ETL ADR 0013 (membership view); docs/reference/insight-catalog/
kbase_sources: [legacy-structure-definition, portal-custom-insights-manage, portal-insights, portal-databases]
last_reviewed: 2026-10-18
---

# Insights and rulesets

## What it is

iDNA scans the code it collects from the focus databases with **insights**. A **pattern** is a regular expression; a **topic** groups one or more patterns; an **insight** consists of one or more topics; a **ruleset** groups insights. An insight matches a code block when every one of its topics matches the block, and a topic matches when any of its patterns does - "OR within a topic, AND across topics". Patterns are PostgreSQL regular expressions applied case-insensitively to each collected code block; many are written per line. Each place where an insight matches is reported as a **finding** (one row per insight, code block and location), shown with the insight's name (`finding_name`) and severity.

## Why it matters

Before suggesting a code search or tooling outside iDNA for "do we have X in our code?", check whether an insight already detects X - its results exist for every analysed application already. [[concepts/shipped-insight-catalog]] lists every shipped insight; [[howtos/find-code-patterns]] is the recipe.

## Details

- **Shipped and custom.** panagenda ships 9 rulesets with 110 insights as of the ETL reference data of 2026-10-18 (commit 1c5d046); a server's live catalog (`cqx_data.design_insight`, `cqx_data.design_insight_ruleset`) may differ with its iDNA build, and for that server it wins. Shipped items are read-only in Custom Insights › Manage; to change one, copy it into a custom item. Customers can add their own patterns, insights and rulesets (`custom = true` in `cqx_data.design_insight` / `cqx_data.design_insight_ruleset`).
- **Enabled.** The nightly run computes findings only for insights of enabled rulesets. An insight in no enabled ruleset produces no findings - no finding is then no proof of clean code.
- **Timing.** New or changed patterns, insights and rulesets take effect with the next nightly design analysis run.
- **Severity scale** (shipped defaults per insight): Insignificant, Very Low, Low, Medium, High, Very High, Exceptional (weights 0, 5, 20, 35, 50, 70, 100); the scale also holds Not Analyzed (-1).
- **The shipped rulesets.**
  - Cloud Readiness (19): what ties an application to its current environment; 13 of its insights repeat a detection of another ruleset.
  - Dependencies on other Databases (3): code that reaches into other Notes databases.
  - Mail Dependencies (11): mail sending, the Domino Directory, calendar and to-do features.
  - Nomad Web, iOS, Android Compatibility (16): a compatibility target, see [[concepts/compatibility-ruleset-routing]].
  - Notes 64bit Readiness (17): a compatibility target, see [[concepts/compatibility-ruleset-routing]].
  - Operating System Dependencies (6): files, directories, other programs, OLE, native calls, the system clock.
  - Special Interfaces (8): ODBC, DB2, the Lotus Connectors, LotusScript extensions, instant messaging, command shells, custom data.
  - Specific Code Findings (19): individual patterns such as hard-coded IP addresses, file names, replica IDs and e-mail addresses, locking, computed code.
  - Web Design (11): browser readiness; "Con:" blocks the web, "Pro:" is web-ready, "Fun:" is informational.
- **Identical detections.** Some insights run exactly the same patterns in two or three rulesets (for example "Hardcoded IP Addresses" in Specific Code Findings and Cloud Readiness). They report the same code: count within one ruleset, or deduplicate by code location; never add the rulesets' counts.
- **Which ruleset an insight belongs to.** On iDNA builds with `cqx_data.design_insight_ruleset_member` (check with `to_regclass('cqx_data.design_insight_ruleset_member')`), that view lists the (ruleset, insight) pairs, including insights without findings; an insight can sit in several rulesets or in none. On older builds the membership shows only through finding rows (`ruleset_id`, `insight_id` in `cqx_data.design_insight_overview_byrepset`).
- **Where results surface.** Design › Insights (treemap by ruleset and finding), the dashboards Insights Findings by Code Element in Database and by Unique Code Block (filters Ruleset and Insight), one Catalog › Databases column set per shipped ruleset, the Application Insights dashboard per replica set, and the MCP finding tools with a `ruleset_id` - see [[howtos/find-design-dashboards]] and [[howtos/find-code-patterns]].
- **What is not analysed.** Only collected design code is scanned; values kept in documents, profile documents or notes.ini are not. Databases flagged "Insights Overview Exclude" carry no findings ([[schema/topics/insights-overview-exclude]]).

## Related

The two compatibility rulesets have their own target profiles and remediation workflow ([[concepts/compatibility-ruleset-routing]]); the other seven rulesets are read through the catalog and the finding surfaces.

## Sources

ETL reference data `dda_ci/080-085` (shipped catalog), `dda_dwh/scheduled/061` (matching per code block, OR within a topic, AND across topics), `dda_dwh/080` (severity scale), ETL ADR 0013 (membership view); kbase pages on Custom Insights (structure and definition, Manage) and the Insights and Databases pages.
