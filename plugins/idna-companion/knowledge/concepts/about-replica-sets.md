---
id: concepts/about-replica-sets
title: Replica Set
type: concept
status: verified
tags: [domain/notes, domain/domino, domain/idna]
related:
  - concepts/replica-id
  - concepts/database-instance
  - concepts/focus-db
  - schema/tables/replica-set
requires_capability: none
source: CoCo Copilot server/knowledge/foundation_domain_knowledge.md
last_reviewed: 2026-04-17
---

# Replica Set

## What it is

A replica is a copy of a database that synchronizes with its peers via Domino replication. All replicas of the same logical application share a single **Replica ID**. The group of replicas sharing that Replica ID is a **Replica Set**.

A replica set represents one **application** in the logical sense. Each replica set typically has multiple `.nsf` instances — the same application deployed on different servers.

## Why it matters

- **Count unique replicaids to count unique applications** — counting NSF files overcounts, because each replica is its own file.
- When the user says "an application," assume they mean a replica set (not a single NSF instance).
- **When the user says "database X," assume the same.** "Database" is the everyday Notes term; "application" and "replica set" are iDNA's vocabulary, not industry standard. A user asking "how many users have used database XYZ?" almost never intends the instance-vs-replica-set distinction — default to replica-set grain (resolve the title to a replicaid) and drop to DB-instance grain only when they distinguish servers or files ("the copy on server A").
- Application-level analysis (usage, remediation, consolidation) almost always operates at the replica-set level.

## Replica-set grain vs DB-instance grain

Most lifecycle decisions live at **replica-set grain** — usage tiers, modernization disposition, stakeholder cohorts. A few decisions drop to **DB-instance grain** — specifically content-age questions tied to a particular `.nsf` file and the archival-instance pick.

| Decision | Grain | Why |
|---|---|---|
| Usage analysis (UAD, tiers, trends) | Replica set | Aggregated across instances by iDNA. |
| Modernization disposition | Replica set | Applications modernize as a unit, not per instance. |
| Stakeholder cohort | Replica set | Departments and locations relate to the application, not a server. |
| Content age (modification timestamps) | Replica set | `doc_modification_*_allreplicas` columns are pre-aggregated across instances. |
| Content age (creation timestamps) | DB instance | `Created (in this file)` is per-instance by definition; see [[concepts/document-age-timestamps]]. |
| Retention configuration & counts | DB instance, then aggregated | Retention is configured per instance via iDNA Custom DB Properties; the lifecycle decision aggregates across instances. |
| Archival-instance pick | DB instance | The whole point is to choose one specific instance. |

Replication divergence across instances within the same set — disagreement on retention configuration, document counts, or modification timestamps — is itself a finding, not noise.

The lifecycle MCP tools follow this grain split: replica-set-grain decisions are served by the candidate/disposition tools (`sunset_candidates` / `sunset_disposition`, `keep_candidates` / `keep_disposition`, `qualify_app_analysis`), which aggregate across instances; DB-instance-grain decisions are served by `archival_instance_pick` and by the `original_vs_derived` layer of `qualify_app_analysis`, which select a specific instance. See [[howtos/use-application-lifecycle-mcp-tools]].

## iDNA data

- Data table: `cqx_data.replica_set`. See [[schema/tables/replica-set]].
- Join key for replica-set-level tables: `replicaid`.
- Key columns include `is_focus_db`, `usage_last90d`, `user_access_days_last90d`, `remediation_classification`, template-inheritance columns, and last-access columns.

## Related

- [[concepts/replica-id]] — the shared identifier.
- [[concepts/database-instance]] — the NSF unit within a set.
- [[concepts/focus-db]] — scoping default applied at this level.

## Sources

CoCo Copilot foundation knowledge.
