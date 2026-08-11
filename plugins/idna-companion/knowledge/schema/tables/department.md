---
id: schema/tables/department
title: cqx_data.department
type: schema
status: verified
tags: [domain/idna, topic/usage, topic/organization]
related:
  - howtos/analyze-usage-by-department
  - schema/tables/usage-cube-repset-vs-department
  - schema/tables/usage-cube-department-summary
  - schema/tables/usage-cube-department-activity-history
  - schema/tables/usage-sessions-by-department
  - schema/topics/department-and-location
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md
last_reviewed: 2026-04-17
---

# cqx_data.department

## Purpose

Base entity for organizational departments. Self-referencing hierarchy: each department can have a parent and a set of children. Used to resolve department IDs to names and navigate org structure. `cqx_data.location` has the same structural shape for physical locations.

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `department_id` | bigint | Primary key. |
| `parent_id` | bigint | FK -> `cqx_data.department.department_id`. Self-reference for hierarchy. |
| `short_name` | text | Abbreviation / code. |
| `name` | text | Full department name. |
| `path` | bigint[] | Full ancestor chain of IDs, from root to this node. |
| `parent_path` | bigint[] | Ancestor chain excluding this node. |
| `parent_shortnames` | text[] | Ancestor short names. |
| `parent_names` | text[] | Ancestor full names. |
| `children_ids` | bigint[] | Direct child department IDs. |
| `children_shortnames` | text[] | Direct child short names. |
| `children_names` | text[] | Direct child full names. |
| `user_count` | numeric | Users assigned to this department. |

## Relationships

- PK: `department_id`.
- `parent_id` -> `cqx_data.department.department_id`.
- `department_id` joins to `usage_cube_repset_vs_department`, `usage_cube_department_summary`, `usage_cube_department_activity_history`, and `usage_sessions_by_department`.

## Typical use

- Resolve department IDs appearing on usage cubes to human-readable names.
- Find all sub-departments of a given org unit via `path @> ARRAY[<ancestor_id>]`.

## Notes and caveats

- Department size may be small in some organizations; iDNA applies threshold-pseudonymization at the ETL layer, so a below-threshold department resolves to a pseudonymized (masked) name here while its usage rows remain present. See [[schema/topics/department-and-location]].
- `cqx_data.location` mirrors this shape for location analysis.

## Sources

CQX schema reference.
