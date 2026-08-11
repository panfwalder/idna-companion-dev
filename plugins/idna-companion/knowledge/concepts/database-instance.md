---
id: concepts/database-instance
title: Database Instance (NSF)
type: concept
status: verified
tags: [domain/notes, domain/domino]
related:
  - concepts/notes-domino-environment
  - concepts/about-replica-sets
  - concepts/replica-id
  - concepts/database-template
  - glossary/nsf
  - schema/tables/database
requires_capability: none
source: CoCo Copilot server/knowledge/foundation_domain_knowledge.md
last_reviewed: 2026-04-17
---

# Database Instance (NSF)

## What it is

An `.nsf` file — the unit of storage in Notes/Domino. Each NSF is a self-contained database holding documents, views, forms, agents, and code (primarily LotusScript or Formula Language; also JavaScript, Java, XPages).

NSF stands for **Notes Storage Facility**.

## Why it matters

- A single environment may contain thousands of NSFs across multiple servers.
- **When the user says "database," assume they mean a single NSF** — a database instance. When the user says "application," assume they mean a replica set (see [[concepts/about-replica-sets]]).
- Database-instance-level analysis is the drill-down view; application-level is typically the default.

## iDNA data

- Data table: `cqx_data.database`. See [[schema/tables/database]].
- Join key for instance-level tables: `database_id`.
- Instance-level usage lives in `cqx_data.usage_summary_by_database`. See [[schema/tables/usage-summary-by-database]].

## Related glossary

- [[glossary/nsf]]

## Sources

CoCo Copilot foundation knowledge.
