---
id: schema/tables/dbviews-used
title: cqx_data.dbviews_used
type: schema
status: verified
tags: [domain/idna, topic/usage, topic/modernization]
related:
  - schema/tables/database
  - schema/topics/cqx-data-execution-surface
  - concepts/database-complexity
  - concepts/focus-db
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md
last_reviewed: 2026-06-22
---

# cqx_data.dbviews_used

## Purpose

One row per **actively-used** view, per database instance. Use this view to answer "which views are actually used in this database / application?" and to drill into whether a finding sits on a view that is never used. It is unnested from `cqx_data.database.database_views -> 'views_used_compact'`.

**"Used" = the view has a built index** (`size_bytes > 0`). This is index existence — evidence the view is materialised and exercised — **not** a record of user opens or session reads. Do not overstate it.

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `replicaid` | text | Replica ID of the application this instance belongs to. FK to [[schema/tables/replica-set]]. |
| `database_id` | integer | The database instance carrying this used view. FK to [[schema/tables/database]]. **Repeats once per used view — not unique on this view.** |
| `name` | text | View name. |
| `alias` | text | View alias. |
| `size_bytes` | bigint | Built view-index size. `> 0` is what marks the view used. |
| `design_document_id` | bigint | Design-note id of the view. Aligns with `cqx_data.code_element.design_document_id`. |

## The replica-set rule (read before excluding anything)

View usage is collected **per database instance**, but complexity and findings are reported at the **replica-set** level, anchored on the *design-reference* instance (the highest-complexity instance — see [[concepts/database-complexity]]). These two grains do not line up:

- A view can be used on one instance of a replica set but **not** on the design-reference instance.
- So the test "is this view used?" for the application must be the **union across all instances**: *used on any instance ⇒ used for the set*.
- Reading only the design-reference instance would wrongly mark a view unused and drop a real finding.

```sql
-- Views used anywhere in the replica set (union across instances)
SELECT DISTINCT name
FROM cqx_data.dbviews_used
WHERE replicaid = '<replicaid>'
ORDER BY name;
```

```sql
-- Views used in one specific instance
SELECT name, alias, size_bytes
FROM cqx_data.dbviews_used
WHERE database_id = <database_id>;
```

## Not focus-gated

Unlike design/complexity analysis (which only runs on Focus DBs — see [[concepts/focus-db]]), view usage is collected across all **licensed** instances. So `dbviews_used` has coverage for applications that are `Not Analyzed` for complexity.

## What this view does not contain

Only **used** views appear here. The full view count, the **unused** count, and per-language code volume live in the aggregate `cqx_data.database.database_views` blob on [[schema/tables/database]]:

- unused views = `count_total - count_used`
- view code sitting on unused views = `count_code - count_code_used`

For enumerating *distinct used views across a set*, matching on `name` alone can mis-count at the margins, because the ETL decides "used" by matching a design view to a built index on **name or alias**; consider `alias` too when the question is an exact distinct-list rather than the binary "used anywhere?" test.

## Related

- [[schema/tables/database]] — the source table; `database_views` carries the aggregate counts.
- [[schema/topics/cqx-data-execution-surface]] — why `cqx_data` is the surface to query first.
- [[concepts/database-complexity]] — the "used vs. present" caveat that this view now answers for views.

## Sources

CQX schema reference; columns verified against the live instance 2026-06-22.
