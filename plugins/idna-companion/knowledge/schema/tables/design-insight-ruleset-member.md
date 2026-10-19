---
id: schema/tables/design-insight-ruleset-member
title: cqx_data.design_insight_ruleset_member
type: schema
status: verified
tags: [domain/idna, topic/modernization]
related:
  - schema/tables/design-insight
  - schema/tables/design-insight-ruleset
  - concepts/insights-and-rulesets
  - howtos/find-code-patterns
requires_capability: none
source: ETL ADR 0013 and its Companion hand-off (iDNA_Applications_ETL/docs/plans/active/design-insight-ruleset-member/companion-handoff.md); deployed and verified on the panagenda test instances 2026-10-18
last_reviewed: 2026-10-18
---

# cqx_data.design_insight_ruleset_member

## Purpose

Which insight belongs to which ruleset: one row per (ruleset, insight) pair, for every insight - with or without findings, shipped or custom, in enabled or disabled rulesets. Available on iDNA builds with ETL ADR 0013; check with `to_regclass('cqx_data.design_insight_ruleset_member')` before use. On older builds the pairs show only through finding rows.

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `ruleset_id` | text | References `cqx_data.design_insight_ruleset.ruleset_id`. |
| `insight_id` | text | References `cqx_data.design_insight.insight_id`. |

## Relationships

- Ruleset side: an inner join to `cqx_data.design_insight_ruleset` loses nothing.
- Insight side: `cqx_data.design_insight` lists only insights with a severity; join from this view with a LEFT JOIN when an insight without severity must show.
- Insights in no ruleset: LEFT JOIN from `cqx_data.design_insight` to this view.

## Typical use

The catalog lookup in [[howtos/find-code-patterns]]: insights by name with their rulesets and the `enabled` state.

## Notes and caveats

- Many-to-many: an insight can sit in several rulesets (several rows) or in none (no row). Shipped content puts each insight in exactly one ruleset; identical detections are separate insights.
- Shipped pairs follow the shipped reference data: when it is reloaded, the pairs of shipped rulesets and of shipped insights are reset to the shipped set, so a customer's assignment that involves a shipped insight or a shipped ruleset does not survive the reload; pairs between custom insights and custom rulesets stay. During a reload the view is briefly incomplete, as the two entity views are.
