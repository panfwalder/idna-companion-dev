---
id: concepts/remediation-category
title: Remediation Category (legacy A/B/C/D → vector-route mapping)
type: concept
status: verified
tags: [domain/idna, topic/modernization]
related:
  - concepts/about-modernization-vector
  - concepts/about-modernization-plan
  - concepts/about-template-inheritance
  - concepts/code-block
  - playbooks/source-code-remediation/classify-template-paths
  - playbooks/source-code-remediation/classify-shared-code-paths
  - playbooks/source-code-remediation/build-remediation-plan
requires_capability: none
source: CoCo_Playbook_64bit_Migration_Consulting.docx; vector-route mapping per the ETL modernization precompute
last_reviewed: 2026-07-24
---

# Remediation Category (legacy A/B/C/D → vector-route mapping)

## What it is

The **historical, manual** four-part classification of how an application with compatibility findings should be fixed, from the pre-vector consulting method. **The operating vocabulary today is the modernization vector's route set** ([[concepts/about-modernization-vector]]) — the ETL precomputes what this scheme derived by hand. This page stays as the mapping between the two, and as the vocabulary of the manual fallback method (custom rulesets / what-if analysis).

## The mapping

| Legacy category | Description | Vector route(s) |
|---|---|---|
| **A** | Clean standard-template association | `template/remediate_master` (estate master) / `template/upgrade_standard` (pristine HCL standard) |
| **B** | Broken explicit link but strong template similarity | `template/investigate_inheritance/similar_master\|similar_standard` (and `template/unify_similar` when the similarity is to unlinked peers rather than a template) |
| **C** | Shared custom code blocks across applications | worklist blocks with reach > 1 (fix-once/reapply; `scope` says whether one sitting covers it) |
| **D** | Unique application code | solo worklist blocks / `direct/remediate_code` |

The vector routes are finer than A/B/C/D in exactly the places that matter: coverage per block (covered/diverging/surplus), the upgrade-don't-edit split, the unscanned-master gate, and the authored-once accounting the categories could only gesture at.

## Why the ordering mattered (and still does)

Customers do not have to attack all findings as isolated fixes — earlier categories remove more findings per unit of effort. That insight survives as the plan's ranking (sessions by apps cleared, blocks by reach) and the three-tier consulting narrative (custom-master multipliers → standard upgrades → piece-work).

The category model answers "where can we fix this most efficiently?" It never answered "what kind of work is this?" — that still comes from the target profile and findings taxonomy (code repair / redesign / conditional execution / dependency follow-up / deployment follow-up).

## Where each path is handled

- Template paths → [[playbooks/source-code-remediation/classify-template-paths]]
- Shared/unique code → [[playbooks/source-code-remediation/classify-shared-code-paths]] and [[playbooks/source-code-remediation/analyze-shared-vs-solo-blocks]]
- Sequencing → [[playbooks/source-code-remediation/build-remediation-plan]]

## Sources

64-bit migration consulting docx (the manual method); ETL modernization precompute (the current vocabulary).
