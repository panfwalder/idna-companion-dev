---
id: playbooks/source-code-remediation/assess-exposure
title: Assess Source-Code Remediation Exposure
type: playbook
status: verified
tags: [domain/idna, topic/modernization, audience/consultant]
related:
  - concepts/about-modernization-vector
  - concepts/about-modernization-plan
  - howtos/query-modernization-views
  - playbooks/source-code-remediation/remediation-routing
  - playbooks/source-code-remediation/triage-working-set
  - concepts/compatibility-ruleset-routing
  - concepts/notes-64bit-compatibility
  - concepts/notes-64bit-findings-taxonomy
  - concepts/notes-64bit-remediation-profile
  - concepts/nomad-web-mobile-compatibility
  - concepts/nomad-web-mobile-findings-taxonomy
  - concepts/nomad-web-mobile-remediation-profile
  - concepts/focus-db
  - howtos/use-source-code-remediation-mcp-tools
  - schema/tables/design-insights-summary-notes64bit
  - schema/tables/design-insights-summary-nomadwebmobile
  - schema/tables/replica-set
requires_capability: none
source: CoCo_Playbook_64bit_Migration_Consulting.docx (manual method) + modernization-vector-integration phase-2 rewrite (consume-first)
last_reviewed: 2026-07-24
---

# Assess Source-Code Remediation Exposure

## When to retrieve

Use when the customer needs the first scoped answer about a compatibility target — either "how ready are we / what breaks" or "how big and severe is the finding volume". These are TWO different questions with two different surfaces; this page teaches when each is the right frame.

## The two lenses

**Readiness (per-app verdict):** "what apps will break on 64-bit clients", "which apps aren't Nomad-ready", "how ready are we". Answer from the precomputed modernization vector — the readiness split (n ready of n kept focus apps) grouped by fix route, via `modernization_plan_overview` (KPI header + session queue) or `modernization_for_app` (one app). Rules that always apply:

- Aggregate through the template lens (routes/sessions) — never a flat affected-app list.
- Findings measure the presence of non-portable code, not guaranteed breakage; a well-maintained app can carry findings for runtime-guarded desktop-only features.
- The vector covers kept (`keep/%`) focus apps only. **An app absent from the vector is out of scope (sunset or non-focus), not implicitly ready** — always say so when a cohort question comes up.
- Readiness is per-target: name the ruleset every answer applies to.
- Close overview answers by offering the drill-down (route lists, session dossier, worklist).

**Exposure (finding volume and severity):** "how many findings, how severe, which apps carry the most". Answer from the exposure tools — `exposure_summary` for the aggregate, `most_affected_apps` for the named working set with usage context. This is the right lens for severity questions and for sizing the problem; it is the WRONG lens for readiness questions (a raw finding wall says nothing about which single session clears what).

## Goal

Convert an intimidating raw count into a picture the customer can reason about — readiness split for "are we ready", application-level exposure for "how big is it".

## Decision logic

- Reframe from database instances to replica sets/applications.
- For exposure: break down by severity, and set the expectation that the count shrinks under triage and template leverage — the plan's session queue is where the shrinkage is already computed.
- Severity means different things across targets: for Notes 64-bit many high findings still lead to code repair or dependency replacement; for Nomad, many comparable findings point to redesign or feature withdrawal.
- Usage context triages, it does not replace the ranking: most estates carry the bulk of finding volume in apps nobody used in 90 days.

## Ask the customer

- Do you want to scope to Focus applications only? Default: yes.
- Readiness or volume — which question is actually being asked? (When in doubt for "how bad is it": give the exposure aggregate AND offer the readiness split.)

## Manual fallback — custom rulesets / what-if analysis

For a ruleset outside the modernization precompute, the exposure queries run directly (privileged sessions; restricted sessions use the tools above or present the SQL for the customer):

Intent: count affected Focus applications and summarize severity.

```sql
SELECT COUNT(DISTINCT dis.replicaid) AS affected_applications,
  SUM(dis.count_findings) AS total_findings,
  SUM(dis.findings_veryhigh) AS very_high,
  SUM(dis.findings_high) AS high,
  SUM(dis.findings_medium) AS medium,
  SUM(dis.findings_low + dis.findings_verylow) AS low_info
FROM <summary_table> dis
JOIN cqx_data.replica_set rs USING (replicaid)
WHERE rs.is_focus_db
```

Intent: list the most-affected applications with usage context.

```sql
SELECT rs.title, dis.count_findings, dis.findings_veryhigh, dis.findings_high,
  dis.findings_medium, rs.user_access_days_last90d, rs.usage_last90d
FROM <summary_table> dis
JOIN cqx_data.replica_set rs USING (replicaid)
WHERE rs.is_focus_db AND dis.count_findings > 0
ORDER BY dis.findings_veryhigh DESC, dis.findings_high DESC, dis.count_findings DESC
LIMIT 20
```

Bind `<summary_table>` from the resolved target profile ([[concepts/compatibility-ruleset-routing]]).

## What to tell the customer

- The meaningful unit is the application/replica set, not the raw NSF count.
- Exposure size does not equal project size — the readiness split and the session queue show how much of it collapses under template leverage.
- This is the entry picture; the plan ([[concepts/about-modernization-plan]]) is the course.

## Output to next task

- readiness split and/or exposure aggregate, per target
- handoff to [[playbooks/source-code-remediation/triage-working-set]] (scope cuts) or straight to the plan queue

## Sources

Consume-first: the ETL-precomputed modernization surfaces. Manual method: 64-bit migration consulting docx, Stage 1.
