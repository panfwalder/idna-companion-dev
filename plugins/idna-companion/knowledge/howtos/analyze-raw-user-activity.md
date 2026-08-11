---
id: howtos/analyze-raw-user-activity
title: Analyze raw per-user activity over any window
type: howto
status: verified
tags: [domain/idna, topic/usage, topic/regulatory, audience/consultant]
related:
  - concepts/usage-time-windows
  - concepts/regulatory-usage-constraints
  - schema/tables/usage-sessions
  - schema/tables/user-activity-summary
  - schema/tables/usage-cube-raw
  - schema/tables/usage-cube-raw-no-summaries
  - schema/tables/usage-cube-repset-vs-user
  - howtos/analyze-usage-over-custom-window
  - howtos/identify-content-creators-and-consumers
requires_capability: mcp_unrestricted
source: docs/reference/user-activity-raw-query.md (validated 2026-07-27)
last_reviewed: 2026-07-27
---

# Analyze raw per-user activity over any window

## Question

"Who used app X between March and June — exactly?" / "What did user N work
with last quarter?" / "Who are the most active users in Sales this year?" —
per-user activity at daily grain, over any window, from any angle. This is
rung 3 of [[concepts/usage-time-windows]]: **privileged profile only**, and
subject to the customer's regulatory posture
([[concepts/regulatory-usage-constraints]]).

## Approach

**The `user_activity_over_window` MCP tool is the primary surface.** Four
pivots, all windowed, all ranked most-active-first:

- `pivot: app` + replicaid — who used one application (incl. the window-exact
  `distinct_users` count and the anonymous share).
- `pivot: user` + username — one user's applications (case-insensitive lookup).
- `pivot: department` / `pivot: location` + id — most active users of one org
  unit, **subtree-inclusive by default** (`include_subtree: false` for direct
  members only).

The tool encodes the join hygiene this page documents: LEFT-JOIN person
resolution (anonymous sessions surface as an explicit
`*** Anonymous / Unresolved ***` bucket, never dropped) and org attribution
from [[schema/tables/user-activity-summary]].

## Data needed

- [[schema/tables/usage-sessions]] — daily person-keyed session facts.
- [[schema/tables/user-activity-summary]] — person_id ↔ username resolution,
  org attribution, person-document flag.
- [[schema/tables/usage-cube-raw]] / [[schema/tables/usage-cube-raw-no-summaries]]
  — weekly per-user cube slices, when weekly grain suffices.

## Procedure — the manual base pattern

One row per (user, application); add the window filter for custom periods.
This is the corrected, validated form (100% of session volume retained):

```sql
SELECT
    username AS "User Name",
    min(department_name) AS "Department",
    min(location_name) AS "Location",
    bool_or(has_person_document) AS "Has Person Document",
    min(db_title) AS "DB Title",
    replicaid AS "Replica ID",
    max(session_date) AS "Last Access Date",
    max(session_date) FILTER (WHERE writes_client > 0) AS "Last Write Date",
    count(DISTINCT session_date) AS "Days of Activity",
    COALESCE(sum(reads_client),0) AS "Document Reads",
    COALESCE(sum(writes_client),0) AS "Document Writes",
    COALESCE(sum(sessions_client),0) AS "Sessions Client",
    COALESCE(sum(sessions_web),0) AS "Sessions Web"
FROM
(
    SELECT
        session_date,
        COALESCE(uas.username, '*** Anonymous / Unresolved ***') AS username,
        COALESCE(department.name, '*** No Department Mapped ***') AS department_name,
        COALESCE(location.name, '*** No Location Mapped ***') AS location_name,
        COALESCE(uas.has_person_document, false) AS has_person_document,
        database_basic.title AS db_title,
        database_basic.replicaid,
        COALESCE(usage_sessions.sessions_client, 0) AS sessions_client,
        COALESCE(usage_sessions.sessions_web, 0) AS sessions_web,
        COALESCE(usage_sessions.reads_client, 0) AS reads_client,
        COALESCE(usage_sessions.writes_client, 0) AS writes_client
    FROM cqx_data.usage_sessions
    JOIN cqx_data.database_basic USING (database_id)
    LEFT JOIN cqx_data.user_activity_summary uas USING (person_id)
    LEFT JOIN cqx_data.department USING (department_id)
    LEFT JOIN cqx_data.location USING (location_id)
    -- Custom time window:
    -- WHERE session_date >= '<from_date>' AND session_date <= '<to_date>'
) act
GROUP BY "User Name", "Replica ID"
ORDER BY "User Name", "DB Title"
```

Angle variations: filter `replicaid` (app pivot), filter `username` (user
pivot), filter department/location id on the inner query (org pivot; subtree
via `department.path @> ARRAY[<id>]` on a join to `cqx_data.department`).

## Interpretation — caveats that change numbers

All verified against a live instance (2026-07-27):

- **LEFT joins are load-bearing.** Inner joins silently drop three
  populations: anonymous web sessions (`person_id IS NULL` — ~20% of *web*
  session volume on the reference instance), users without person-document org
  mapping, and — under an eval/trial license — everyone (dep/loc IDs are
  NULLed wholesale in `user_activity_summary`).
- **The anonymous bucket is bucket-grain**: many unattributable actors collapse
  into one row per app; its "Days of Activity" is not per-person. It is
  excluded from distinct-user counts — say so when reporting them.
- **Residual undercount:** sessions recorded under a user's non-primary name
  variations or secondary node IDs do not resolve to the user (the summary
  keeps one node ID per username) and land in the anonymous bucket.
- **Servers appear as users.** Domino servers replicating or accessing
  databases show up with large session counts and no person document. Check
  suspicious heavy "users" against `cqx_data.server.notes_name` before naming
  them in a report — and note that servers from foreign domains still look
  like users.
- **One department/location per user, by construction** — a current-snapshot
  attribution applied to all historical sessions (`min()` in the ETL build),
  which is why `min(department_name)` in the pattern is safe.
- **State the coverage when reporting an org pivot** — subtree-inclusive is the
  default, so say what the figures cover ("Sales plus N sub-departments" /
  "Sales only, direct members"). The tool reports the subtree size; relay it —
  on estates where the org unit has children, this sentence is load-bearing.
- **Eval-license mechanisms differ by table:** `user_activity_summary` NULLs
  the org *IDs*; the usage cube masks *names* while IDs remain.

## Variations

- **Weekly grain suffices** — use [[schema/tables/usage-cube-raw-no-summaries]]
  (leaf grain, session metrics) or [[schema/tables/usage-cube-raw]] with all
  five dimensions pinned (adds UAD); mind the grouping-set warning on that page.
- **All-time totals per (app, user)** — [[schema/tables/usage-cube-repset-vs-user]]
  is the pre-aggregated shortcut; the recipe around it is
  [[howtos/identify-content-creators-and-consumers]].
- **Aggregate-only customer** — this page does not apply; stay on
  [[howtos/analyze-usage-over-custom-window]] and
  [[howtos/attribute-usage-aggregate-only]].

## Sources

Maintained source: `docs/reference/user-activity-raw-query.md` (dev repo) —
derived from a production Metabase pattern, forensically corrected and
validated 2026-07-27 (result contains exactly 100% of `usage_sessions` volume).
