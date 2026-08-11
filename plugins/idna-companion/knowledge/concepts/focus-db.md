---
id: concepts/focus-db
title: Focus DB and Focus Replica Set
type: concept
status: verified
tags: [domain/idna]
related:
  - concepts/about-replica-sets
  - concepts/database-instance
  - concepts/database-complexity
  - schema/tables/replica-set
  - schema/tables/database
requires_capability: none
source: CoCo Copilot server/knowledge/foundation_domain_knowledge.md
last_reviewed: 2026-04-17
---

# Focus DB and Focus Replica Set

## What it is

iDNA flags the most important database instances as **Focus DBs**: those whose `database_type` is either `"Application"` or `"Mail-in DB"`. This excludes system databases, legacy mail files, and other noise that would otherwise dominate aggregate statistics.

A **Focus Replica Set** is a replica set that consists of Focus DB instances.

## Default behavior

Unless the user explicitly directs otherwise, **limit analysis to Focus DBs / Focus Replica Sets** — apply the filter `WHERE is_focus_db`.

Always state clearly in the answer when this filter is applied.

## When to override

- The user explicitly asks about non-focus databases (system, archive, scratch).
- Listing master templates — templates are **not** Focus DBs, so `is_focus_db` must be left off. See [[concepts/database-template]].
- A specific question requires a broader environment view (e.g., total instance counts regardless of classification).

## iDNA columns

- `is_focus_db` (boolean) on `cqx_data.replica_set` and `cqx_data.database`.

## Sources

CoCo Copilot foundation knowledge (Analysis Guidelines + Focus DBs section).
