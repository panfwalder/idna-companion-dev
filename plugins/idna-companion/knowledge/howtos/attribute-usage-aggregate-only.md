---
id: howtos/attribute-usage-aggregate-only
title: Attribute application usage at aggregate grain only
type: howto
status: verified
tags: [domain/idna, topic/usage, topic/regulatory, audience/consultant]
related:
  - howtos/analyze-usage-by-department
  - concepts/regulatory-usage-constraints
  - schema/topics/department-and-location
requires_capability: none
source:
last_reviewed: 2026-05-04
---

# Attribute application usage at aggregate grain only

## Question

"Who uses this application?" — at a customer where per-user / per-session attribution is prohibited (works council, EU AI Act, customer policy). The answer must rank departments and locations and describe activity volume without naming individuals or exposing per-session detail.

This is the **strict-mode** counterpart to [[howtos/analyze-usage-by-department]]. Use that one when the customer has signed off on per-user attribution; use this one otherwise. The framing for which mode applies is in [[concepts/regulatory-usage-constraints]].

## Approach

Stay inside the cube/summary table family on [[schema/topics/department-and-location]]. iDNA already applies threshold-pseudonymization to those tables at the ETL layer: below-threshold departments and locations appear with masked names, rows intact. Drop down to the session tables only if the customer has explicitly authorised it — and at strict customers they will not have.

The application-level rank itself (UAD per replica set) is *not* per-user — `cqx_data.replica_set.user_access_days_last90d` is safe in both modes. The strict/permissive split bites at the attribution step, not at the ranking step.

## Data needed

- [[schema/topics/department-and-location]] — the cube/summary family (covers app x department, app x location, per-department summary, per-location summary, weekly activity-history twins).
- [[schema/tables/replica-set]] — application-level UAD windows (`user_access_days_last90d` / `_last365d` / `_all_time`) and aggregate `users_active_last90d` count. Aggregate-grain by definition.

## Procedure

### Variant A — "Which departments use application X?" (strict-safe)

1. Resolve the application's `replicaid`.
2. Query the cube cross-reference; project only aggregate columns.

   Intent: rank departments that accessed the given application by UAD, exposing only group-level metrics.

   ```sql
   SELECT rs.title, c.department_name,
     c.user_access_days, c.users_active, c.last_access
   FROM cqx_data.replica_set rs
   JOIN cqx_data.usage_cube_repset_vs_department c USING (replicaid)
   WHERE rs.replicaid = '<replicaid>'
   ORDER BY c.user_access_days DESC NULLS LAST
   ```

3. State the pseudonymization caveat: low-volume departments appear under masked names per iDNA's ETL threshold rule — the rows and counts remain, only the name is pseudonymized.

### Variant B — "Where (locations) is application X used?"

Switch dimension to `cqx_data.usage_cube_repset_vs_location`. Column substitutions per [[schema/topics/department-and-location]]: `location_id`, `location_name`, `locations_active`.

### Variant C — "What is department Y's overall activity profile?"

Use `cqx_data.usage_cube_department_summary` (per-department aggregate across all applications). No per-user columns; safe under strict mode.

### Variant D — "Is department Y's activity rising or falling?"

Use `cqx_data.usage_cube_department_activity_history` (weekly per-department UAD trend). Aggregate-grain only.

## Interpretation

- **`users_active` is the count of distinct users**, not their identities. Reporting "12 users in the Finance department" is aggregate; reporting *which* 12 is not.
- **`user_access_days`** combines users x days at the cube grain — see [[concepts/user-access-days]].
- **Masked-name rows** still carry real usage counts; only the department/location name is pseudonymized below iDNA's threshold. Report the usage, but state that the cohort cannot be attributed to a named org unit — a masked name is not zero usage and not a missing row. See [[schema/topics/department-and-location]].
- **`last_access` without `last_access_user`** is the strict-safe recency signal at this grain. The user-attributed last-access columns (`last_accessed_user`, `last_write_user` on `replica_set` / `usage_summary_by_database`) must not be queried under strict mode.

## What drops out under strict mode

Do **not** run these at a strict customer:

- **Per-user cubes and summaries** — `cqx_data.usage_cube_repset_vs_user`, `cqx_data.usage_cube_user_summary`, `cqx_data.usage_cube_user_activity_history`, `cqx_data.repset_activity_summary_per_user`, `cqx_data.person_activity_summary`, `cqx_data.user_activity_summary`. All carry `username` (or join trivially to `cqx_data.person`).
- **Raw session detail** — `cqx_data.usage_sessions` (keyed on `person_id`), and the unfiltered `cqx_data.usage_cube_raw` / `cqx_data.usage_cube_raw_no_summaries`.
- **Last-user columns naming individuals** — `last_accessed_user`, `last_write_user` on `cqx_data.replica_set` and `cqx_data.usage_summary_by_database`; `db_last_accessed_user`, `db_last_write_user`, `rs_last_accessed_user`, `rs_last_write_user` on `cqx_data.database_content_retention_summary`.
- Any aggregate query that filters or groups by a single user (e.g. joining `cqx_data.person` and grouping by `person_id`).

Note: `cqx_data.usage_sessions_by_department` and `cqx_data.usage_sessions_by_location` are **not** per-user — they are session counts keyed on `(database_id, server_id, department_id|location_id, session_date)` with no `username`. They are strict-safe and may be queried for per-department / per-location daily session activity.

Under restricted mode (`access_profile = 'restricted'`), `execute_sql` is unavailable entirely — the runtime profile gate refuses it before any SQL is issued (it is a privileged-only tool). The MCP server's Postgres role additionally denies the listed tables at the database layer per [[concepts/regulatory-usage-constraints]], so even the privileged-only path could not read them in restricted mode. Under permissive mode (`access_profile = 'privileged'`), `execute_sql` is available, the platform allows the query, and the boundary is a customer-policy concern in the SQL you write.

## Variations

- **Permissive customer** — switch to [[howtos/analyze-usage-by-department]] and bring per-user columns into scope.
- **Sub-tree of an org** — `cqx_data.department.path @> ARRAY[<ancestor_id>]` works the same in strict mode; the path itself is aggregate.

## Sources

Companion runtime constraint, framed in [[concepts/regulatory-usage-constraints]]. Schema family documented in [[schema/topics/department-and-location]].
