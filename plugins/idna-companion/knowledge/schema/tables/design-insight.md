---
id: schema/tables/design-insight
title: cqx_data.design_insight
type: schema
status: verified
tags: [domain/idna, topic/modernization]
related:
  - schema/tables/design-insight-ruleset
  - schema/tables/design-insight-ruleset-member
  - schema/tables/design-insight-overview-byrepset
  - concepts/insights-and-rulesets
  - concepts/shipped-insight-catalog
  - howtos/find-code-patterns
requires_capability: none
source: ETL repo cqx_data_ifa/010 (view definition); columns verified on the panagenda test instance 2026-10-18
last_reviewed: 2026-10-18
---

# cqx_data.design_insight

## Purpose

The live catalog of insights on this iDNA server: one row per insight, shipped and custom. Use it to find an insight by name, to see its severity, and to list the customer's own insights. It carries no ruleset; the membership is in [[schema/tables/design-insight-ruleset-member]].

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `insight_id` | text | Insight identifier; `insight_id` in the finding views. |
| `name` | text | Insight name; equals `finding_name` in the finding views. |
| `description` | text | Description; for many shipped insights it repeats the name. |
| `custom` | boolean | `true` for customer-defined insights, `false` for shipped ones. |
| `owner_name` | text | `panagenda` for shipped insights. |
| `severity` | text | Severity name (Insignificant ... Exceptional). |
| `severity_weight` | integer | Numeric severity for ordering (0 ... 100). |

## Relationships

- `insight_id` -> `cqx_data.design_insight_ruleset_member.insight_id` (one insight can sit in several rulesets or in none).
- `insight_id` -> `cqx_data.design_insight_overview_byrepset.insight_id` / `design_insight_overview_bydb.insight_id`.

## Typical use

Find insights by name: see the catalog lookup in [[howtos/find-code-patterns]]. The customer's own insights: `WHERE custom`.

## Notes and caveats

- The view joins the severity scale with an inner join: an insight without a severity is not listed here, although it can appear in the membership view.
- Identical detections exist as separate insights with their own ids (e.g. "Hardcoded IP Addresses" in two rulesets) - see [[concepts/insights-and-rulesets]].
- The shipped content differs between iDNA builds; for one server this view is the truth, [[concepts/shipped-insight-catalog]] is the reference snapshot.
