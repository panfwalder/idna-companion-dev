---
id: howtos/analyze-usage-by-department
title: Analyze application usage by department
type: howto
status: verified
tags: [domain/idna, topic/usage, topic/organization]
related:
  - concepts/user-access-days
  - concepts/about-replica-sets
  - schema/tables/replica-set
  - schema/tables/department
  - schema/tables/usage-cube-repset-vs-department
  - schema/topics/department-and-location
  - howtos/attribute-usage-aggregate-only
  - concepts/regulatory-usage-constraints
requires_capability: none
source: CoCo Copilot server/knowledge/topic_department_location.md
last_reviewed: 2026-04-17
---

# Analyze application usage by department

## Question

"Which departments use application X?" — or the mirror: "What applications does department Y use?"

## Approach

Use the pre-aggregated cross-reference cube `cqx_data.usage_cube_repset_vs_department`. Join to `replica_set` (for application context) and/or `department` (for org hierarchy). State the privacy caveat upfront: iDNA pseudonymizes (masks) the names of below-threshold departments — the rows and their counts remain present.

## Data needed

- [[schema/tables/usage-cube-repset-vs-department]] — the app × department cube.
- [[schema/tables/replica-set]] — for application context (title, Focus flag).
- [[schema/tables/department]] — for hierarchy navigation if needed.
- Background: [[schema/topics/department-and-location]] (the location twin follows the same shape).

## Procedure

### Variant A — "Which departments use this application?"

1. Resolve the application's `replicaid` (from title or user context).
2. Run the cube query.

   Intent: list departments that have accessed the given application, ranked by UAD.

   ```sql
   SELECT rs.title, d.department_name, d.user_access_days,
     d.users_active, d.last_access
   FROM cqx_data.replica_set rs
   JOIN cqx_data.usage_cube_repset_vs_department d USING (replicaid)
   WHERE rs.replicaid = '<replicaid>'
   ORDER BY d.user_access_days DESC NULLS LAST
   ```

### Variant B — "What applications does this department use?"

1. Resolve `department_id` from `cqx_data.department`.
2. Flip the filter.

   Intent: list Focus applications accessed by the given department, ranked by UAD.

   ```sql
   SELECT rs.title, d.user_access_days, d.users_active, d.last_access
   FROM cqx_data.usage_cube_repset_vs_department d
   JOIN cqx_data.replica_set rs USING (replicaid)
   WHERE d.department_id = <department_id>
     AND rs.is_focus_db
   ORDER BY d.user_access_days DESC NULLS LAST
   ```

## Interpretation

- **Masked department names** — iDNA applies threshold-pseudonymization at the ETL layer. A department whose user count falls below iDNA's configured threshold appears with its name pseudonymized (masked); the row and its usage counts stay present. Rows are not dropped — a below-threshold cohort is a masked-name row, not a missing one. State this to the user. See [[schema/topics/department-and-location]].
- **`users_active` vs `user_access_days`** — active user count is distinct users; UAD accumulates users × days. Combine for a full picture (e.g. "5 users, 120 UAD" means heavy repeat usage).
- **`last_access`** in the distant past on a high-UAD row — usage was meaningful historically but has dropped.

## Variations

- **By location instead of department** — switch to `usage_cube_repset_vs_location`. Column substitutions: `location_id`, `location_name`, `locations_active` ↔ `department_id`, `department_name`, `departments_active`.
- **Time trend** — use `usage_cube_department_activity_history` (weekly per department).
- **All apps summary per department** — use `usage_cube_department_summary`.
- **Sub-tree of an org** — use `cqx_data.department.path @> ARRAY[<ancestor_id>]` to include all descendants.

## Sources

CoCo Copilot department/location reference.
