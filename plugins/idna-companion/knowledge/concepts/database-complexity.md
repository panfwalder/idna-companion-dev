---
id: concepts/database-complexity
title: Database Complexity
type: concept
status: draft
tags: [domain/idna, topic/modernization]
related:
  - concepts/design-similarity
  - concepts/focus-db
  - concepts/usage-tier
  - concepts/remediation-classification
  - schema/tables/replica-set
  - schema/tables/database
  - schema/tables/dbviews-used
  - schema/tables/replica-set-summary
  - schema/tables/design-summary-by-replica-set
  - schema/topics/complexity
  - glossary/business-critical
requires_capability: none
source: docs/reference/database-complexity-reviewed.md
last_reviewed: 2026-06-22
---

# Database Complexity

## What it is

A weighted, computed technical score that iDNA derives from a database's design and source code. It answers one question: *how difficult is this application likely to be to understand, modernise, migrate, remediate, or redevelop?* It is **not** usage, business criticality, or content-retention risk — it is the technical design dimension, and its value comes from being read **alongside** those other signals.

Complexity is a **comparative KPI**: it is good for ranking databases against each other ("which of my apps are the heavy ones?"). It is a relatively blunt instrument — see the redirect rule below before using it to drive a project decision.

## Redirect rule (apply this when answering complexity questions)

When a user asks "which are my most complex applications?", surface and read the metric — but **do not present flat complexity as a project-navigation signal.** Always pivot to the user's actual challenge and to the more actionable signals:

- For a migration/remediation project, the scoped finding scores (e.g. `cqx_data.design_insights_summary_notes64bit.vulnerability_score`) are far more actionable than flat complexity — they say *what* breaks and *where*.
- Flat complexity is noisy for the reasons below (template inflation, dormant elements, analysis scope), so offer to go deeper rather than ending at a complexity ranking.

A good answer looks like: *"Read flat complexity like this; it's calculated from design and code volume. But tell me the challenge you're solving — for most decisions I can give you something more useful per database than the raw complexity number, because of A, B, C."*

## How it is derived

Complexity is computed from collected design data — design elements (forms, views, agents, script libraries, fields, …), code blocks, and code lines — and is **weighted, not a flat count**. Examples of factors that influence the weighting include code language type, Reader/Author fields, custom XPages controls, hard-coded references, Notes-client and local-system dependencies, mail/third-party/encryption dependencies, and migration-relevant code patterns. Factors are considered both individually and **in combination**: when several occur together, the rating can be higher than the parts individually.

> This is not a published scoring specification. The factor list above is illustrative, not complete or ranked; do not infer weights, completeness, or thresholds from it.

## Grain: the instance is the source

Complexity is computed at the **database instance** — `cqx_data.database.design_complexity_score` (numeric) is where the score originates, one per analysed NSF on a server. The **replica set / application** value (`cqx_data.replica_set.design_complexity_score`) is **not an aggregate or sum across replicas**: it is **inherited** from a single reference instance, `database_id_design_reference`, which is the **highest-complexity instance** in the set (verified equal to that instance's score) — the same anchoring iDNA uses for replica-set findings.

So when the question is *what makes this app complex*, drill to the instance via [[schema/tables/database]] and the per-language volumes in [[schema/tables/design-summary-by-replica-set]]; the replica-set number is just the reference instance's score surfaced at application grain. The instance table carries only the raw numeric score — the categorical grade below is assigned at the replica-set / summary level.

## Grade labels

The observed grade labels are **Very High / High / Medium / Low / Very Low / Not Analyzed** (the simplified family collapses the middle to *Medium / High* and *Low / Very Low*). These are observed dashboard/database labels, not a documented universal taxonomy, and they are a **distinct label family from usage tiers** — usage tiers' sixth label is *No Usage*; complexity's is *Not Analyzed*. The categorical complexity tier used by [[concepts/remediation-classification]] is this same metric read at a coarser grain.

## Computed vs. manually-entered — do not confuse

The computed score is the `design_complexity_*` family. The columns `complexity_estimation` on `cqx_data.replica_set` / `cqx_data.database` are a different thing: they sit in the customer-annotation block beside `effort_estimation` / `cost_estimation` / `business_owner` and are sparsely populated, which indicates a **manually-entered planning field, not the computed complexity score** (inference, not a documented fact). Never read `complexity_estimation` as iDNA's calculated complexity.

## Why the raw number must be read carefully

1. **Template inflation.** Complexity scores the *entire* design, including dormant elements inherited from the template a database was built on. Mail-template-derived apps especially (also Discussion, TeamRoom, Document Library) carry large inherited bulk that inflates the number without representing custom work. The correction is to read complexity through **design similarity**: high complexity *with* a standard-template relation is mostly inherited; high complexity *without* one is genuinely custom, where effort and risk concentrate. See [[concepts/design-similarity]].
   - **Grain rule:** the application-level template-relation signal lives on `cqx_data.replica_set` (`design_is_similar_to_standard_template`, `most_similar_template_*`). The `cqx_data.design_insight_overview_bydb` columns (`suspected_standard_template_relation`, `similar_standard_templates`) are per-instance/per-finding **drill-down evidence** to aggregate by `replicaid` — never a single instance's flag taken as the application verdict.
2. **Analysis scope (focus-DB gating).** Only Focus DBs and templates are analysed for design and code, so only they have a complexity score. Most of the estate is therefore **Not Analyzed** — and *no score does not mean a simple app*, it usually means the app was never analysed. See [[concepts/focus-db]].
3. **Used vs. present.** Neither complexity nor findings tell you which elements are actually *used* — but for views you can now check. **View-index usage is available per database instance** via [[schema/tables/dbviews-used]] (used = a built view index), with aggregate counts in `cqx_data.database.database_views`; an environment-wide rollup also exists in `cqx_data.environment_kpis` (`views_defined` / `views_indexed`) but is no longer the only source. Usage is per instance, so for an application-level read aggregate across the replica set's instances (used on any instance ⇒ used). Per-Form document counts still come from iDNA **content analysis** (`cqx_data.database_content_analysis.form_details`), which is a heavy, separately-licensed, per-database scan — a deliberate instrument for a few triaged apps, not a casual step.

## Where it appears in the product

- Design → Complexity — `{{idna_server}}/#/complexity`: ranked complexity index of focus DBs, with standard-template reference lines for orientation.
- Remediation Overview — `{{idna_server}}/metabase/dashboard/10000005`: user activity plotted against design complexity (the coarse tier behind `remediation_classification`).

## Related

- [[concepts/design-similarity]] — the correction lens for template inflation.
- [[concepts/remediation-classification]] and [[glossary/business-critical]] — where the coarse complexity tier feeds triage.
- [[schema/topics/complexity]] — the full cross-table field map.

## Sources

`docs/reference/database-complexity-reviewed.md` (reviewed compendium), grounded against the live `cqx_data` schema 2026-06-15.
