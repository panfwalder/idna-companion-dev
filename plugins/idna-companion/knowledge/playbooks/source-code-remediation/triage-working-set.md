---
id: playbooks/source-code-remediation/triage-working-set
title: Triage the Source-Code Remediation Working Set
type: playbook
status: verified
tags: [domain/idna, topic/modernization, topic/usage, audience/consultant]
related:
  - concepts/about-modernization-plan
  - playbooks/source-code-remediation/remediation-routing
  - playbooks/source-code-remediation/assess-exposure
  - playbooks/source-code-remediation/classify-template-paths
  - playbooks/source-code-remediation/classify-shared-code-paths
  - concepts/compatibility-ruleset-routing
  - concepts/notes-64bit-remediation-profile
  - concepts/nomad-web-mobile-remediation-profile
  - concepts/remediation-classification
  - concepts/user-access-days
  - concepts/focus-db
  - schema/tables/replica-set
  - schema/tables/design-insights-summary-notes64bit
  - schema/tables/design-insights-summary-nomadwebmobile
  - playbooks/application-lifecycle/lifecycle-routing
  - playbooks/application-lifecycle/sunset-funnel
  - playbooks/source-code-remediation/build-remediation-plan
  - schema/tables/dbviews-used
  - concepts/regulatory-usage-constraints
requires_capability: none
source: CoCo_Playbook_64bit_Migration_Consulting.docx
last_reviewed: 2026-07-24
---

# Triage the Source-Code Remediation Working Set

## When to retrieve

Use when the customer asks which affected applications really need remediation now, or when the agent needs to stop the conversation from jumping prematurely into code-fix discussions.

## What the precompute already does — and what it does not

The modernization vector already applies the biggest scope cut: it covers **kept (`keep/%`) focus apps only** — sunset apps have no vector rows and no plan sessions (they still appear as design-family peers and in estate-wide reach: evidence, not work). The plan queue already ranks the remaining work by apps cleared. What the precompute deliberately does NOT do is the **business overlay**: criticality the customer supplies, retention obligations on low-usage apps, and the conscious keep/retire dispositions. That overlay is this page's job.

Two triage moves remain live:

1. **Lifecycle scope-cut cross-check** — apps falling out of the remediation working set as retire/archive candidates get their disposition from the sunset funnel ([[playbooks/application-lifecycle/sunset-funnel]] / `sunset_disposition`), never inferred from a usage query. The lifecycle boundary is authoritative: lifecycle decides keep-vs-sunset; modernization works within keep.
2. **Business overlays on the plan** — the customer names business-critical apps and retention constraints; those reorder or annotate the plan queue, they do not recompute it.

## Goal

Split the affected application set into:

- active applications that proceed to remediation analysis
- inactive applications that can be retired, archived, deferred, or consciously kept in scope for retention reasons

## Decision logic

The core consulting rule: "unused" is not the same as "disposable."

Classify each application into:

- active
- inactive and a clear retirement candidate
- inactive but requiring retention review

For the retention-review bucket, the customer must choose a conscious disposition rather than leaving the question unresolved.

Note: this rule is at the **application** level. There is a separate, **element-level** refinement — excluding findings that sit on *unused views* — which belongs in the remediation plan, not in this retire/keep triage. See [[playbooks/source-code-remediation/build-remediation-plan]] (and [[schema/tables/dbviews-used]] for the per-view evidence).

## Ask the customer

- What usage horizon should we emphasize in the discussion: 3, 6, or 12 months?
- Are there legal, regulatory, or business-retention requirements for low-usage applications?
- Are there applications that must remain usable even if they are rarely opened?

## Manual fallback — custom rulesets / what-if analysis

Bind `<summary_table>` from the resolved target profile.

Intent: cross-reference findings with current usage and recency (privileged sessions; restricted sessions present the SQL or stay with the tool surfaces).

```sql
SELECT rs.title, dis.count_findings, dis.findings_veryhigh, dis.findings_high,
  rs.usage_last90d, rs.remediation_classification,
  rs.user_access_days_last90d, rs.last_accessed
FROM <summary_table> dis
JOIN cqx_data.replica_set rs USING (replicaid)
WHERE rs.is_focus_db AND dis.count_findings > 0
ORDER BY rs.user_access_days_last90d DESC NULLS LAST
```

## Retention options to present

- archive on a dedicated server with legacy access if needed
- flag and defer as known incompatible for the target platform/runtime
- keep in the active set because it remains business-critical when needed

When an application falls out of the remediation working set as a retire/archive candidate, the actual delete-vs-archive disposition (retention precondition, archival-instance pick, scan-freshness confidence) is produced by the sunset funnel — hand off to [[playbooks/application-lifecycle/sunset-funnel]] / `sunset_disposition` rather than inferring the disposition from the usage query above.

## What to tell the customer

- This task prevents wasted remediation effort.
- Retention is a business decision, not something the skill should silently infer from usage alone.
- Some low-usage applications may still become cheap remediation candidates later if they are template-based.

## Output to next task

- active working set with business overlays
- deferred set with documented disposition (via the sunset funnel)
- handoff to the plan queue (`modernization_plan_overview`) or, for manual classification, [[playbooks/source-code-remediation/classify-template-paths]] and [[playbooks/source-code-remediation/classify-shared-code-paths]]

## Open methodological limits from the docx

- The docx asks for consulting consensus on recommended inactivity thresholds; it does not establish one authoritative cutoff yet.

## Sources

Consume-first: the ETL-precomputed modernization surfaces (keep-only cohort). Manual method: 64-bit migration consulting docx, Stage 2.
