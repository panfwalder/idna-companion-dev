---
id: schema/tables/database-content-retention-summary
title: cqx_data.database_content_retention_summary
type: schema
status: verified
tags: [domain/idna, topic/content-age, audience/consultant]
related:
  - schema/tables/database-content-retention
  - schema/tables/database
  - schema/tables/replica-set
  - schema/tables/usage-summary-by-database
  - concepts/user-access-days
  - playbooks/application-lifecycle/sunset-funnel
  - concepts/document-age-timestamps
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md
last_reviewed: 2026-05-04
---

# cqx_data.database_content_retention_summary

## Purpose

Per-database-instance summary of document age and retention status for scanned NSFs. One row per `database_id`. Combines retention configuration (`retention_years`, `retention_mode`, `retention_cutoff`), document-age aggregates (creation/modification timestamps, `doc_age_months_avg`), in-vs-out-of-retention document counts, and pre-joined recency and UAD windows at both DB-instance and replica-set grain.

This is the **default** retention page. Use it for almost every retention question. Consult [[schema/tables/database-content-retention]] only when a monthly distribution around the cutoff is needed for evidence narratives — it carries no columns the summary does not already expose at coarser grain.

**MCP-first for lifecycle work.** For application-lifecycle / sunset decisions, the Companion calls the semantic lifecycle tools (`sunset_candidates`, `sunset_disposition`, `archival_instance_pick`) rather than querying this table directly — see [[playbooks/application-lifecycle/sunset-funnel]] and [[howtos/use-application-lifecycle-mcp-tools]]. This page is the **column-and-arithmetic reference layer behind those tools** (e.g. the retention-horizon and content-age-scan columns the tools aggregate and flag), and the reference for ad-hoc retention SQL outside the lifecycle funnels.

## Prerequisite

Retention values come from iDNA Custom DB Properties; without that configuration `retention_years` / `retention_mode` / `retention_cutoff` are NULL across the estate and `docs_count_inret` is meaningless. Check `retention_cutoff` coverage before running retention-locked queries — a NULL cutoff covers both unconfigured retention *and* configured-but-incomplete cases (e.g. `retention_mode` set but `retention_years` missing, so ETL never produced a cutoff).

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `database_id` | bigint | PK. FK to `cqx_data.database`. |
| `replicaid` | text | FK to `cqx_data.replica_set`. |
| `title` | text | Database title. |
| `server_name` | text | Server hosting this instance. |
| `file_name` | text | NSF path. |
| `replicas_count` | integer | Replicas in the set. |
| `database_type` | text | Database-type label. |
| `is_licensed` | boolean | iDNA licensing flag. |
| `is_content_age_analyzed` | boolean | Whether content-age analysis has run for this DB. |
| `collection_status` | text | Collection state for the content-age scan. |
| `contentage_scantime` | timestamp tz | When the content-age scan ran. |
| `contentage_scanduration_seconds` | numeric | Scan duration. |
| `docs_count_total` | bigint | Total documents in the NSF. |
| `docs_count_catalog` | bigint | Catalog/system documents excluded from retention math. |
| `doc_age_months_avg` | numeric | Average document age, months. |
| `retention_years` | integer | Configured retention period (years). |
| `retention_mode` | text | Retention configuration mode. NULL when not configured. |
| `retention_cutoff` | date | Boundary date: documents modified after this are inside retention. |
| `docs_count_inret` | bigint | Documents inside the retention period. **Primary lock signal for the dormants playbook.** |
| `docs_count_noret` | bigint | Documents outside retention. |
| `docs_pct_inret` | numeric | Share of documents inside retention. |
| `db_last_accessed` | date | Most recent **read** at the DB-instance grain. Dormancy signal. |
| `db_last_accessed_user` | text | Last reader at instance grain. |
| `db_last_write_access` | date | Most recent **write** at the DB-instance grain. Contradicts dormant status when recent. |
| `db_last_write_user` | text | Last writer at instance grain. |
| `rs_last_accessed` | date | Most recent read across the whole replica set. |
| `rs_last_accessed_user` | text | Last reader across the set. |
| `rs_last_write_access` | date | Most recent write across the whole replica set. |
| `rs_last_write_user` | text | Last writer across the set. |
| `doc_creation_earliest_inthisfile` | timestamp tz | Earliest document creation seen in this file. |
| `doc_creation_latest_inthisfile` | timestamp tz | Latest document creation seen in this file. |
| `doc_modification_earliest_allreplicas` | timestamp tz | Earliest document modification across all replicas. |
| `doc_modification_latest_allreplicas` | timestamp tz | Latest document modification across all replicas. Use with `retention_years` to project the retention horizon when `retention_cutoff` is NULL. |
| `db_users_active_last30d` / `_last90d` / `_alltime` | integer | Distinct active users at instance grain. |
| `db_user_access_days_last30d` / `_last90d` / `_alltime` | integer | UAD at instance grain. |
| `rs_users_active_last30d` / `_last90d` / `_alltime` | integer | Distinct active users at replica-set grain. |
| `rs_user_access_days_last30d` / `_last90d` / `_alltime` | integer | UAD at replica-set grain. |

## Relationships

- PK: `database_id`.
- `database_id` -> `cqx_data.database.database_id`.
- `replicaid` -> `cqx_data.replica_set.replicaid`.
- The pre-joined `db_*` and `rs_*` UAD/recency columns mean most retention questions can be answered without joining `cqx_data.usage_summary_by_database` or `cqx_data.replica_set` directly. Fall back to [[schema/tables/usage-summary-by-database]] only when per-instance recency or session-level detail not exposed here is needed (e.g., write-session counts).

## Typical use

Intent: list dormant applications still holding documents inside the retention period, ranked by retention-protected document count.

```sql
SELECT s.title, s.server_name, s.replicaid,
  s.docs_count_inret, s.docs_pct_inret, s.retention_cutoff,
  s.rs_last_accessed, s.rs_last_write_access,
  s.rs_user_access_days_last90d
FROM cqx_data.database_content_retention_summary s
WHERE s.docs_count_inret > 0
  AND COALESCE(s.rs_user_access_days_last90d, 0) = 0
ORDER BY s.docs_count_inret DESC NULLS LAST
LIMIT 50
```

Intent: estimate the retention horizon for an unconfigured-cutoff case using the latest document modification plus the configured retention years.

```sql
SELECT title, replicaid,
  doc_modification_latest_allreplicas,
  retention_years,
  (doc_modification_latest_allreplicas + (retention_years || ' years')::interval)::date AS retention_horizon
FROM cqx_data.database_content_retention_summary
WHERE retention_mode IS NOT NULL AND retention_cutoff IS NULL
```

## Notes and caveats

- **`docs_count_inret > 0` is the retention-lock signal**, not `retention_years > 0`. An app can be configured for retention yet hold zero in-retention documents, in which case it is not retention-locked.
- **Read vs write recency are different signals.** `db_last_accessed` / `rs_last_accessed` measure reads (the dormancy signal). `db_last_write_access` / `rs_last_write_access` measure writes — recent writes contradict dormant status even if no one is reading.
- **DB grain vs replica-set grain.** `db_*` columns describe one NSF; `rs_*` columns aggregate across the whole replica set. For *usage* questions use replica-set grain — an application is not unused unless the entire replica set is unused.
- **Retention is per-DB only.** `retention_years`, `retention_mode`, `retention_cutoff`, `docs_count_inret`, `docs_pct_inret`, and the `doc_*` timestamp columns are computed per `database_id`; they have no replica-set-grain twin. With healthy Domino replication, those values should match across replicas of the same set. The convention for lifecycle work is to **pick one reference DB instance per replica set** (customer choice, or fall back to `cqx_data.replica_set.database_id_design_reference`) rather than aggregate. Material divergence between replicas is itself a finding worth surfacing.
- This table exposes UAD windows at 30d / 90d / all-time only; the 365d window referenced by retirement-grade analysis lives on [[schema/tables/replica-set]] (`user_access_days_last365d`).
- **NULL `retention_mode` means retention has not been configured for this DB.** NULL `retention_cutoff` covers a broader case: both unconfigured (no `retention_mode`) *and* configured-but-incomplete (`retention_mode` set without `retention_years`, so ETL never produced a cutoff). For retention-aware queries, filter `retention_cutoff IS NOT NULL` — that gate excludes both cases cleanly. Use `retention_mode IS NOT NULL AND retention_cutoff IS NULL` as a diagnostic to surface incomplete configurations.
- Architectural context: this view sits on the public `cqx_data` query surface. See [[schema/topics/cqx-data-execution-surface]].

## Sources

CQX schema reference. Retention values are populated from iDNA Custom DB Properties, which the customer configures per database.
