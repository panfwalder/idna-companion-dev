---
id: concepts/replica-id
title: Replica ID
type: concept
status: verified
tags: [domain/notes, domain/domino]
related:
  - concepts/about-replica-sets
  - concepts/database-instance
requires_capability: none
source: CoCo Copilot server/knowledge/foundation_domain_knowledge.md
last_reviewed: 2026-04-17
---

# Replica ID

## What it is

The unique identifier shared by every replica of a Notes/Domino database. All database instances that replicate with each other carry the same Replica ID.

The Replica ID does **not** change per instance — that is the definition of belonging to the same replica set.

## Why it matters

- **Replica ID is the application-level key.** Joining iDNA tables at the application level always joins on `replicaid`.
- Counting distinct `replicaid` values gives a count of applications, not NSF files. See [[concepts/about-replica-sets]] for the distinction.
- If something claims replica IDs change per instance, that is wrong — likely a confusion with the database instance ID (which is per-NSF).

## iDNA data

- Column `replicaid` (text) appears on `cqx_data.replica_set` and on many application-level iDNA tables.
- See [[schema/tables/replica-set]].

## Related

- [[concepts/about-replica-sets]] — the group keyed by this ID.
- [[concepts/database-instance]] — the per-NSF unit, identified separately by `database_id`.

## Sources

CoCo Copilot foundation knowledge.
