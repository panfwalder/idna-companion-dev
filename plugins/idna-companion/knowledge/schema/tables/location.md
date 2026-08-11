---
id: schema/tables/location
title: cqx_data.location
type: schema
status: verified
tags: [domain/idna, topic/usage, topic/organization]
related:
  - schema/tables/department
  - schema/tables/usage-cube-repset-vs-location
  - schema/tables/usage-cube-location-summary
  - schema/tables/usage-cube-location-activity-history
  - schema/tables/usage-sessions-by-location
  - schema/topics/department-and-location
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md
last_reviewed: 2026-04-17
---

# cqx_data.location

## Purpose

Base entity for organizational locations. Self-referencing hierarchy: each location can have a parent and a set of children. Used to resolve location IDs to names and navigate the location tree. `cqx_data.department` is the structurally equivalent department twin.

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `location_id` | bigint | Primary key. |
| `parent_id` | bigint | FK -> `cqx_data.location.location_id`. Self-reference for hierarchy. |
| `short_name` | text | Abbreviation / code. |
| `name` | text | Full location name. |
| `path` | bigint[] | Full ancestor chain of IDs, from root to this node. |
| `parent_path` | bigint[] | Ancestor chain excluding this node. |
| `parent_shortnames` | text[] | Ancestor short names. |
| `parent_names` | text[] | Ancestor full names. |
| `children_ids` | bigint[] | Direct child location IDs. |
| `children_shortnames` | text[] | Direct child short names. |
| `children_names` | text[] | Direct child full names. |
| `user_count` | numeric | Users assigned to this location. |

## Relationships

- PK: `location_id`.
- `parent_id` -> `cqx_data.location.location_id`.
- `location_id` joins to `usage_cube_repset_vs_location`, `usage_cube_location_summary`, `usage_cube_location_activity_history`, and `usage_sessions_by_location`.

## Typical use

- Resolve location IDs appearing on usage cubes to human-readable names.
- Find all sub-locations of a given node via `path @> ARRAY[<ancestor_id>]`.

## Notes and caveats

- Location size may be small in some organizations; iDNA applies threshold-pseudonymization at the ETL layer, so a below-threshold location resolves to a pseudonymized (masked) name here while its usage rows remain present. See [[schema/topics/department-and-location]].
- `cqx_data.department` mirrors this shape for department analysis.

## Sources

CQX schema reference.
