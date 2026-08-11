---
id: playbooks/application-lifecycle/lifecycle-routing
title: Application Lifecycle Playbook Overview
type: playbook
status: verified
tags: [domain/idna, topic/consolidation, audience/consultant]
related:
  - playbooks/application-lifecycle/sunset-funnel
  - playbooks/application-lifecycle/keep-funnel
  - playbooks/application-lifecycle/stakeholder-cohort
  - playbooks/source-code-remediation/remediation-routing
  - howtos/use-application-lifecycle-mcp-tools
  - howtos/find-most-used-applications
  - concepts/regulatory-usage-constraints
  - concepts/user-access-days
  - concepts/usage-tier
  - concepts/about-replica-sets
  - concepts/focus-db
  - concepts/remediation-classification
  - concepts/document-age-timestamps
  - schema/tables/replica-set
requires_capability: none
source:
last_reviewed: 2026-05-21
---

# Application Lifecycle Playbook Overview

## When to retrieve

Use when the customer arrives with a lifecycle question — phrases like *"what are my unused databases?"*, *"help us consolidate our Domino estate"*, *"which apps can we retire?"*, *"how do we get rid of Notes?"* The trigger is the **shape of the estate**, not a specific compatibility target. Compatibility-driven remediation work belongs in [[playbooks/source-code-remediation/remediation-routing]].

**Also enter here for modernization-prep prioritization questions** — *"which DBs are most important to prepare for 64-bit?"*, *"what should I check before I roll out new clients?"*, *"make sure everything works on the new clients"*. These read like a *"most used"* ranking but carry a forward target / rollout goal, so they are lifecycle-driven triage, **not** the plain UAD ranking in [[howtos/find-most-used-applications]]. The discriminator is a compatibility/migration **target or rollout event** (64-bit, Nomad, new clients, client rollout, migration, upgrade); a usage phrasing with no such cue stays on the UAD path.

## Data-currency preflight

Lifecycle bucketing depends on iDNA's usage tiers and `remediation_classification` being current. **Before walking either funnel, check `usage_data_freshness` from `get_idna_session`.** If the most recent `last_accessed` date is older than 7 days, raise this warning to the consultant:

> *iDNA usage data has not been updated in the last 7 days. Lifecycle bucketing depends on current collection data — recommendations may be invalidated. Verify iDNA collection status before proceeding.*

**If `usage_data_freshness` is `null`** (no usage data has landed yet — typically a freshly-onboarded tenant), treat it the same as the >7-days warning: lifecycle bucketing depends on data that does not exist yet and recommendations are not trustworthy. Verify iDNA collection has run before proceeding.

The content-age scan-freshness gate in [[playbooks/application-lifecycle/sunset-funnel]] catches one specific staleness case (sunset/delete decisions). This preflight catches the broader case where the bucket partition itself is based on stale inputs.

## Goal

Every Focus replica set carries an ETL-computed `lifecycle_vector` (`<direction>/<disposition>[/<reason>]`, see [[concepts/lifecycle-vector]]); the tools read it (non-focus → NULL, not classified). Bring each app to a near-binary recommendation:

| Disposition              | Meaning                                                              | Worked in                                         |
| ------------------------ | ------------------------------------------------------------------- | ------------------------------------------------- |
| `sunset/archive`         | retention configured consistently, documents in retention           | [[playbooks/application-lifecycle/sunset-funnel]] |
| `sunset/delete`          | retention configured consistently, nothing in retention             | [[playbooks/application-lifecycle/sunset-funnel]] |
| `sunset/qualify/*`       | **blocked** — no / conflicting retention config or replication conflict; resolve before disposing | [[playbooks/application-lifecycle/sunset-funnel]] |
| `keep/modernize`         | `usage_last90d >= 'Medium'`                                          | [[playbooks/application-lifecycle/keep-funnel]]   |
| `keep/qualify/usage_low` | low 90d usage (residual) — per-app diagnosis                         | [[playbooks/application-lifecycle/keep-funnel]]   |
| `keep/*/conflict_*`      | kept, with a config/replication conflict to resolve (qualifier)     | [[playbooks/application-lifecycle/keep-funnel]]   |

The consultant's job is to bring each app as close to a **binary yes/no recommendation** as possible. The customer's input on the qualify bucket should collapse to per-app yes/no, not *"tell us your strategy."*

## Tool entry point

This playbook family runs on the semantic lifecycle MCP tools (see [[howtos/use-application-lifecycle-mcp-tools]] for the full map). The scoping entry point is **`lifecycle_exposure_summary`** — it returns the per-disposition histogram (`sunset_archive`, `sunset_delete`, `sunset_qualify`, `keep_modernize`, `keep_qualify`) across the estate plus the context signals this page's preflight depends on (`usage_data_freshness`, retention-configuration coverage, `source_code_access_available`). Call it first when the customer arrives with the naive *"which apps can we retire?"* question, then route into the funnels:

- sunset cohort → `sunset_candidates` → `sunset_disposition` (+ `archival_instance_pick`)
- keep cohort → `keep_candidates` → `keep_disposition` / `qualify_app_analysis`

The Companion calls these tools and interprets their returns. The base bucket is the ETL column `cqx_data.replica_set.lifecycle_vector`, which also serves as a degraded SQL fallback if the MCP is down (base bucket only — see [[concepts/lifecycle-vector]] and the fallback rule in [[howtos/use-application-lifecycle-mcp-tools]]).

## Operating mode

**Restricted is the default.** Stakeholder language stays at department/location grain. Per-user attribution is permissive-only and lives in linked howtos, not here. See [[concepts/regulatory-usage-constraints]] for the mode framing and the MCP-side enforcement model.

## Routing

- *"Which apps can we retire?"* / *"What are my unused databases?"* / *"Which dormant apps still hold legal content?"* → [[playbooks/application-lifecycle/sunset-funnel]]
- *"What about everything else?"* / *"Which apps need modernization?"* / *"How do we prepare for 64-bit or Nomad?"* → [[playbooks/application-lifecycle/keep-funnel]]
- *"Who depends on this app?"* / *"Which departments use it?"* → [[playbooks/application-lifecycle/stakeholder-cohort]] (cross-cutting overlay used by both funnels)

## Modernization-prep: work both ends inward

For a modernization-prep question (above), the estate is a spectrum: the most valuable apps at one end, the disposable ones at the other, with the hard decisions in the middle. A good answer works **both ends inward** and shows traction early. This **augments** — does not replace — the sunset-vs-keep direction fork above; plain consolidation questions still pick a direction.

Two key points the funnels otherwise leave implicit:

- **Sunset = rollout-scope reduction = quick traction.** Every app retired is one you never have to test or remediate for the new clients. Surfacing sunset candidates is part of a modernization-prep answer, not a separate errand — retiring them shrinks the rollout surface before any fix.
- **Match breadth to the question's framing:**
  - **Narrow** (*"which DBs are most important to prep for 64-bit?"*) → lead **keep-end**: `keep_candidates(target=...)` for the `keep/modernize` cohort, with 64-bit findings as the rollout-risk overlay. Then *offer* the sunset scope-reduction / full triage as a follow-up (`AskUserQuestion`) rather than dumping it.
  - **Broad** (*"we're doing a 64-bit migration, how do I prepare my environment?"*) → open with `lifecycle_exposure_summary` (the estate shape, no target needed), then present prep-valuable (`keep/modernize`) and retire-disposable (`sunset`) together, deferring the `keep/qualify` middle.

## Defaults set here, not duplicated in funnels

- **Scope:** Focus replica sets (`is_focus_db = true`) per [[concepts/focus-db]]. Override path: customer can request a wider scope; state the scope in the answer.
- **Operating mode:** restricted by default.

## Deferred to keep-funnel section 1

The customer's **forward target** (HCL Notes 64-bit / Nomad Web / Nomad Mobile / multiple / none-yet) is asked at the entry to [[playbooks/application-lifecycle/keep-funnel]], not here. Customers who only want sunset work should not be asked the target question.

## Required inputs

- The customer has a lifecycle / consolidation question (not a compatibility one).
- The `usage_data_freshness` preflight has passed, or the warning has been surfaced.
- Retention configuration in iDNA Custom DB Properties is in place if sunset is in scope (otherwise the delete-vs-archive split collapses — see sunset funnel for the precondition gate).

## Required foundations

- [[concepts/about-replica-sets]] — applications are replica sets; lifecycle decisions live at replica-set grain (with content-age and archival pick dropping to instance grain).
- [[concepts/focus-db]] — default scope filter.
- [[concepts/usage-tier]] — tier names and the 90d canonical column.
- [[concepts/remediation-classification]] — the `Consolidation Potential` flag is the sunset gate.
- [[concepts/regulatory-usage-constraints]] — restricted vs permissive mode.

## Output

Entry into one or both funnels with scope and operating mode bound. Per-app bucket decisions are produced inside the funnel pages, not here.

## Sources

Workflow design from `docs/plans/active/app-lifecycle-playbook-rewrite/app-lifecycle-playbook-rewrite.md`. Replaces an earlier `lifecycle-routing.md` that was structured around three retrievable phases rather than the four-bucket model.
