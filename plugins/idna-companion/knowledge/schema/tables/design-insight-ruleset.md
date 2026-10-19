---
id: schema/tables/design-insight-ruleset
title: cqx_data.design_insight_ruleset
type: schema
status: verified
tags: [domain/idna, topic/modernization]
related:
  - schema/tables/design-insight
  - schema/tables/design-insight-ruleset-member
  - concepts/insights-and-rulesets
  - concepts/compatibility-ruleset-routing
  - howtos/find-code-patterns
requires_capability: none
source: ETL repo cqx_data_ifa/010 (view definition); columns verified on the panagenda test instance 2026-10-18
last_reviewed: 2026-10-18
---

# cqx_data.design_insight_ruleset

## Purpose

The rulesets on this iDNA server: one row per ruleset, shipped and custom, with the switch that decides whether the nightly run evaluates it.

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `ruleset_id` | text | Ruleset identifier; `ruleset_id` in the finding views and the MCP finding tools. Shipped rulesets have the same id on every server ([[howtos/find-code-patterns]]). |
| `name` | text | Ruleset name; `ruleset_name` in the finding views. |
| `description` | text | Description; for most shipped rulesets it repeats the name. |
| `custom` | boolean | `true` for customer-defined rulesets. |
| `enabled` | boolean | `true` = evaluated by the nightly run; a disabled ruleset produces no findings. |
| `owner_name` | text | `panagenda` for shipped rulesets. |

## Relationships

- `ruleset_id` -> `cqx_data.design_insight_ruleset_member.ruleset_id` (its insights).
- `ruleset_id` -> `cqx_data.design_insight_overview_byrepset.ruleset_id` (its findings).

## Typical use

List the rulesets with their state, or bind a `ruleset_id` for the MCP finding tools. The modernization layer evaluates only the configured rulesets (`cqx_data.design_insight_ruleset_modernization`, without an `enabled` filter) that are also enabled.

## Notes and caveats

- Notes 64bit Readiness and Nomad Web, iOS, Android Compatibility are compatibility targets with their own routing ([[concepts/compatibility-ruleset-routing]]).
