---
id: schema/topics/department-and-location
title: Usage by Department and Location
type: schema
status: verified
tags: [domain/idna, topic/usage, topic/organization]
related:
  - concepts/user-access-days
  - schema/topics/cqx-data-execution-surface
  - schema/tables/department
  - schema/tables/location
  - schema/tables/usage-cube-repset-vs-department
  - schema/tables/usage-cube-repset-vs-location
  - schema/tables/usage-cube-department-summary
  - schema/tables/usage-cube-department-activity-history
  - schema/tables/usage-cube-location-summary
  - schema/tables/usage-cube-location-activity-history
  - schema/tables/usage-sessions-by-department
  - schema/tables/usage-sessions-by-location
  - schema/tables/replica-set
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md
last_reviewed: 2026-04-17
---

# Usage by Department and Location

## Purpose

Department and location usage analysis follows the same schema shape: tables come in department/location pairs. This page documents the pairing so you can switch dimensions by substituting table and column names rather than changing the analysis pattern.

## The table families

| Use case | Department table | Location equivalent |
|---|---|---|
| App x org-unit cross-reference | [[schema/tables/usage-cube-repset-vs-department]] | [[schema/tables/usage-cube-repset-vs-location]] |
| Per-org-unit summary (all apps) | [[schema/tables/usage-cube-department-summary]] | [[schema/tables/usage-cube-location-summary]] |
| Weekly trend per org unit | [[schema/tables/usage-cube-department-activity-history]] | [[schema/tables/usage-cube-location-activity-history]] |
| Raw session detail | [[schema/tables/usage-sessions-by-department]] | [[schema/tables/usage-sessions-by-location]] |
| Base entity (hierarchy, names) | [[schema/tables/department]] | [[schema/tables/location]] |

Column substitutions when switching dimensions:

- `department_id` <-> `location_id`
- `department_name` <-> `location_name`
- `departments_active` <-> `locations_active`

## Table selection principle

Pre-aggregated first. Prefer the cube cross-reference, summary, and activity-history tables over the raw session tables. Drop to `usage_sessions_by_*` only when the cubes do not expose the grain you need.

These objects are still part of the public `cqx_data` execution surface, so they are the preferred starting point for department/location analysis even when the underlying derivation logic lives upstream. See [[schema/topics/cqx-data-execution-surface]].

## Privacy

iDNA applies threshold-pseudonymization at the ETL layer. When `pseudonymize=Y`, a department or location whose user count falls below iDNA's configured threshold has its **name pseudonymized (masked)** at the source — but the row stays present with its usage counts intact. Below-threshold cohorts surface as rows with a masked name, not as missing or suppressed rows. The `department` and `location` lookup tables that resolve IDs to names carry the same threshold mechanism. For `pseudonymize=N` tenants, real names appear under the customer's pseudonymization-waiver attestation. State to the user that small org units may appear under masked names rather than treating absent names as missing data.

## Typical questions

- "Which departments use application X?" -> `usage_cube_repset_vs_department` filtered by `replicaid`
- "What applications does department Y use?" -> `usage_cube_repset_vs_department` filtered by `department_id`
- "How is department Y's activity trending?" -> `usage_cube_department_activity_history`
- "Which locations access this application?" -> `usage_cube_repset_vs_location`

## Sources

CQX schema reference.
