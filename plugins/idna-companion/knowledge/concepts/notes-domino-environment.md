---
id: concepts/notes-domino-environment
title: Notes/Domino Environment
type: concept
status: verified
tags: [domain/notes, domain/domino]
related:
  - concepts/database-instance
  - concepts/about-replica-sets
requires_capability: none
source: CoCo Copilot server/knowledge/foundation_domain_knowledge.md
last_reviewed: 2026-04-17
---

# Notes/Domino Environment

## What it is

An organization's deployment of HCL Notes (client) and HCL Domino (server). Environments accumulate databases over decades — scope ranges from a handful of NSF files to tens of thousands spread across multiple Domino servers.

## Why it matters

Scale is the defining characteristic. Consulting engagements (consolidation, migration, modernization) become meaningful only when the analysis layer (iDNA) can span the full estate. Individual database analysis does not scale; application-level and aggregate views do.

## Key shape

- **Database instances** (`.nsf` files) are the unit of storage. See [[concepts/database-instance]].
- **Applications** are logical entities — a replica set of instances sharing a `replicaid`. See [[concepts/about-replica-sets]].
- **Templates** (`.ntf`) define shared design, enabling fix-once-propagate-many remediation. See [[concepts/database-template]].
- **Usage** is measured at both the instance and application level; application-level is typically the default. The primary way to measure usage are "User Access Days" (UAD) in the last 90 days. See [[concepts/user-access-days]].

