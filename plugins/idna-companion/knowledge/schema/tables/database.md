---
id: schema/tables/database
title: cqx_data.database
type: schema
status: verified
tags: [domain/idna, topic/usage, topic/modernization]
related:
  - concepts/database-instance
  - concepts/database-template
  - concepts/about-replica-sets
  - concepts/focus-db
  - concepts/about-template-inheritance
  - concepts/database-complexity
  - schema/topics/complexity
  - schema/topics/cqx-data-execution-surface
  - schema/tables/usage-summary-by-database
  - schema/tables/replica-set
  - schema/tables/usage-sessions
  - schema/tables/dbviews-used
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md
last_reviewed: 2026-06-22
---

# cqx_data.database

## Purpose

One row per licensed database instance: the per-server, per-file view of NSF/NTF files. Use this table when the question is about a specific server, file path, instance-level ownership or custom properties, or master templates. For application-level analysis use [[schema/tables/replica-set]] instead.

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `database_id` | integer | Primary key for the instance. Join key for per-instance tables. |
| `replicaid` | text | Replica ID linking this instance to its replica set / application. |
| `title` | text | Database title. |
| `title_unique` | text | Disambiguated title string. |
| `server_name` | text | Server hosting this instance. |
| `server_id` | bigint | FK to `cqx_data.server`. |
| `file_name` | text | On-server file path. |
| `database_type` | text | `"Application"`, `"Mail-in DB"`, `"Mail"`, `"Template"`, etc. |
| `size_bytes` | bigint | Database size. |
| `documentcount` | bigint | Document count for the instance. |
| `design_complexity_score` | numeric | **Per-instance design-complexity score — this is the source grain for complexity** (the replica-set value inherits the reference instance's score, not a sum). See [[concepts/database-complexity]]. |
| `design_rank` | bigint | Rank of this instance by design complexity. |
| `insights_score` | numeric | Per-instance code-insights score. |
| `similarity_score` | numeric | Per-instance design-similarity score. |
| `complexity_estimation` | text | **Manually-entered** planning estimate (customer-annotation block, sibling of `effort_estimation`/`cost_estimation`), *not* the computed `design_complexity_score`. |
| `is_focus_db` | boolean | Focus flag for application/mail-in instances. |
| `is_template_candidate` | boolean | iDNA flag indicating likely template behavior. |
| `template_inherits_from` | text | Explicit parent template link for this specific instance. |
| `template_acts_as_master` | text | If set, this database serves as a master template. |
| `usage_summary` | jsonb | JSON summary blob for usage metrics. |
| `design_summary` | jsonb | JSON summary blob for design metrics. |
| `insights_summary` | jsonb | JSON summary blob for code-insight metrics. |
| `custom_properties` | jsonb | JSON blob of custom DB properties. |
| `status` | text | Lifecycle/status field from portfolio governance. |
| `disposition` | text | Disposition / target-state field. |
| `classification` | text | Classification field. |

## Relationships

- PK: `database_id`.
- `replicaid` -> `cqx_data.replica_set.replicaid`.
- `server_id` -> `cqx_data.server.server_id`.
- `database_id` joins to `cqx_data.usage_summary_by_database`, `cqx_data.usage_sessions_by_department`, and `cqx_data.usage_sessions_by_location`.
- Master templates are discovered here: `WHERE template_acts_as_master IS NOT NULL`.

## Typical use

- Listing master templates (templates are typically not Focus DBs, so do not filter `is_focus_db`):
  ```sql
  SELECT title, template_acts_as_master, server_name, file_name
  FROM cqx_data.database
  WHERE template_acts_as_master IS NOT NULL
  ```
- Per-server instance inventories.
- Drill-down from an application (replica set) to specific NSF instances on specific servers.

### Visual surface

For one specific instance, the iDNA per-DB-instance detail component surfaces virtually all collected information at `{{idna_server}}/#/replicadetails/<database_id>`. Useful as a drill-down whenever analysis lands on a particular `database_id` — for example after an archival-instance pick or a replication-divergence finding. No equivalent detail page exists for replica sets; for application-level visual entry use the Replica Set Catalog at `{{idna_server}}/#/masterlistreplicasets`.

## Notes and caveats

- `template_inherits_from` here is a single text value. On `replica_set` the equivalent is a `text[]` because one replica set can contain instances inheriting from different templates.
- **Complexity originates here.** `design_complexity_score` is computed per instance. The application-level value on `cqx_data.replica_set` is **inherited** from the highest-complexity instance in the set (`replica_set.database_id_design_reference`) — verified equal to that instance's score — not an aggregate across replicas. The categorical grade (Very High … Not Analyzed) is assigned at the replica-set / `replica_set_summary` level; this table carries only the raw numeric score. See [[concepts/database-complexity]] and [[schema/topics/complexity]].
- This table includes several JSON summary columns (`database_properties`, `usage_summary`, `design_summary`, `insights_summary`, `misc_properties`, `custom_properties`) that are useful for dashboarding but are not ideal as primary query surfaces.
- **`database_views` (jsonb) — per-instance view aggregate.** Unlike the blobs above, this one is genuinely useful as an answer surface. It carries view counts — `count_total`, `count_used` (views with a built index = **used**), `count_code` (views carrying code), `count_code_used` (carries code **and** used) — plus per-language code-block (`num_cb_*`) and lines-of-code (`num_loc_*`) totals (`*_f` formula, `*_ls` LotusScript, `*_js` JavaScript, `*_j` Java), and a `views_used_compact[]` array of the used views. Derived signals: **unused views** = `count_total - count_used`; **view code on unused views** = `count_code - count_code_used`. The used-view array is exposed row-per-view as [[schema/tables/dbviews-used]] — prefer that for listing used views. Counts here are **per instance**; for an application-level read, aggregate across the replica set's instances.
- Templates are routinely excluded by `is_focus_db`; always lift that filter when working with NTFs. See [[concepts/database-template]].
- Architectural context: the ETL/code wiki describes `cqx_data` as the main public execution surface for CQX/DataMiner users. See [[schema/topics/cqx-data-execution-surface]].

## Sources

CQX schema reference.
