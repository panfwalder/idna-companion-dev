---
id: concepts/code-block
title: Code Block
type: concept
status: verified
tags: [domain/idna, topic/modernization]
related:
  - concepts/remediation-category
  - schema/tables/code-source
  - schema/tables/design-insight-overview-byrepset
  - playbooks/source-code-remediation/classify-shared-code-paths
requires_capability: none
source: CoCo_Playbook_64bit_Migration_Consulting.docx + CoCo seed material
last_reviewed: 2026-04-17
---

# Code Block

## What it is

A unit of source code identified by iDNA via a content hash. The same normalized block may appear in many different applications.

## Why it matters

The docx treats the code-block lens as one of the main efficiency insights of the engagement: fixing one high-impact block can resolve findings across many applications at once.

## iDNA data

- finding-level references: [[schema/tables/design-insight-overview-byrepset]]
- normalized source lookup: [[schema/tables/code-source]]

## Playbook use

The shared-code and unique-code analysis is handled in [[playbooks/source-code-remediation/classify-shared-code-paths]].

## Sources

64-bit migration consulting docx and CoCo seed material.
