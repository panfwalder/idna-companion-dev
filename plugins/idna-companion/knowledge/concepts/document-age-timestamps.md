---
id: concepts/document-age-timestamps
title: Document Age Timestamps
type: concept
status: verified
tags: [domain/notes, domain/domino, topic/content-age]
related:
  - concepts/about-replica-sets
  - schema/tables/database-content-retention-summary
  - playbooks/application-lifecycle/sunset-funnel
  - playbooks/application-lifecycle/keep-funnel
requires_capability: none
source:
last_reviewed: 2026-05-21
---

# Document Age Timestamps

## What it covers

Every Notes document carries multiple timestamps recording when it was created and modified. The distinction between *"initially"* and *"in this file"* variants is operationally important — different timestamps answer different questions about the same document.

## The four timestamps

| Timestamp | Meaning |
|---|---|
| `Created (initially)` | When the document was first created, anywhere in the replica set. |
| `Created (in this file)` | When the document arrived in *this particular* database instance — either by manual creation or by being replicated in. |
| `Modified (initially)` | When the most recent modification was first performed, anywhere. |
| `Modified (in this file)` | When that modification reached *this particular* instance. |

An `(initially)` variant equals its `(in this file)` counterpart only when the operation was performed in this very instance.

## When to use which

- **Content age** → `Modified (initially)` / `doc_modification_earliest_allreplicas`. Tells you how old the content actually is across replicas, regardless of when replication propagated it.
- **Archival-instance pick** → earliest `Created (in this file)` among **surviving instances** of the replica set. This identifies the best archival target — the instance documents arrived in first. It is **not provably the historical original**: older instances may have been decommissioned from the estate, leaving the chosen instance as the oldest *survivor* rather than the absolute origin.
- **Replication health check** → divergence in modification timestamps across replicas of the same set. Healthy replication should produce convergent `Modified (in this file)` timestamps within a small window; large divergence is itself a finding.

## Schema mapping

Aggregated forms exposed in iDNA tables include:

- `doc_modification_earliest_allreplicas` / `doc_modification_latest_allreplicas` on [[schema/tables/database-content-retention-summary]] — minimum and maximum modification timestamps across replicas of the set.
- `doc_creation_earliest_inthisfile` on the same table — earliest creation-in-this-file for the instance.

Column-level details live on the schema page.

## Common pitfalls

- **Confusing *"in this file"* with *"initially"*.** Equal only when the document was created or modified in this very instance. In most cases they differ.
- **Treating creation timestamp as a content-age signal.** Creation timestamps are *provenance* signals — when did this document arrive here? Content age is a *modification* question — when was the content last touched?
- **Assuming replicas should have identical `(in this file)` timestamps.** They legitimately differ across replicas based on when replication reached each instance.
- **Treating the earliest-creation instance as *the* original.** It is the original among **surviving** replicas. Older instances may have been deleted from the estate, making the chosen instance a best-available proxy rather than a proof of historical origin.
- **Treating content-age data as always-fresh.** Content-age data is scan-derived (typically every second Saturday). Stale or failed scans (`Collection Status` = Error / Warning, old `Scan Time` in the Retention Catalog) yield content-age signals that no longer reflect reality. Always validate before acting on `sunset/delete` candidates.

## Related

- [[concepts/about-replica-sets]] — the unit these timestamps describe across instances.
- [[schema/tables/database-content-retention-summary]] — the aggregated timestamp columns and the table's grain.

## Sources

Notes/Domino document metadata semantics. Citation to be added — to be sourced from Notes/Domino administration documentation or CoCo Copilot foundation knowledge; tracked in `docs/plugin/todo.md`.
