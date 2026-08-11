---
id: concepts/remediation-classification
title: Remediation Classification
type: concept
status: verified
tags: [domain/idna, topic/consolidation]
related:
  - concepts/about-replica-sets
  - concepts/user-access-days
  - concepts/database-complexity
  - schema/tables/replica-set
  - playbooks/source-code-remediation/triage-working-set
  - playbooks/application-lifecycle/lifecycle-routing
  - concepts/lifecycle-vector
requires_capability: none
source: CoCo Copilot server/knowledge/foundation_domain_knowledge.md + CoCo_Playbook_64bit_Migration_Consulting.docx + Remediation Overview dashboard ({{idna_server}}/metabase/dashboard/10000005) and product-owner input
last_reviewed: 2026-06-12
---

# Remediation Classification

## What it is

iDNA's application-level triage signal. The column `remediation_classification` on `cqx_data.replica_set` assigns each application a category from a rough combination of two dimensions: **user activity** and **design complexity** (`design_complexity_score`).

It is surfaced to customers via the Remediation Overview dashboard (`{{idna_server}}/metabase/dashboard/10000005`), where applications are plotted on user activity (x) against design complexity (y). The intent is to give a customer facing a blank page a starting point for triage — not a definitive judgement. Every value here is a rough estimation.

## Key value: Consolidation Potential

The most operationally useful value:

- **`"Consolidation Potential"`** - no or very low usage in recorded history and no usage in the recent working window

Applications flagged as Consolidation Potential are strong candidates for retirement, archiving, or defer-and-review decisions.

## Secondary classifications

- **`"Business Critical"`** (usage ≥ High, complexity ≥ Medium) - applications with at least high usage and at least medium complexity often turn out to be integral parts of the customer's business processes.
- **`"Quick Win Candidate"`** (usage ≥ Low, complexity ≤ Low) - candidates for re-development (modernize / migrate) to reach a broad user base: they have usage, and comparatively little effort should be required to re-develop them.

## Other values (mention only)

- **`"No Design Data"`** - no design data was collected, so complexity cannot be measured and the other classifications cannot be computed.
- **`"Expert Opinion"`** - the middle ground between Quick Win and Business Critical; hard to determine automatically, so it calls for expert judgement.

## How it informs the playbook

The working-set triage step uses this signal to help identify applications that should not automatically proceed into remediation work. See [[playbooks/source-code-remediation/triage-working-set]].

## Caveat

"No recent usage" is not the same as "not needed." Retention and business criticality still require explicit customer decisions. All of these bins are rough estimations and a starting point — not final decisions.

## Sources

CoCo seed material plus the 64-bit migration consulting docx.
