---
id: schema/tables/replica-set-summary
title: cqx_data.replica_set_summary
type: schema
status: verified
tags: [domain/idna, topic/usage, topic/consolidation, topic/modernization]
related:
  - schema/tables/replica-set
  - concepts/database-complexity
  - concepts/usage-tier
  - concepts/remediation-classification
  - concepts/lifecycle-vector
  - schema/topics/complexity
requires_capability: none
source: docs/reference/cqx_data_schema_enriched_all_tables.md
last_reviewed: 2026-06-19
---

# cqx_data.replica_set_summary

## Purpose

A compact per-replica-set rollup of the headline usage, complexity, and remediation KPIs — each as a label plus an ordinal index for sorting. It is the lightweight companion to the wide `cqx_data.replica_set` table: use it when you want the categorical grades and their sort indices without selecting the full application row. Scoped to analysed applications (≈600 rows on the POC, vs. the full replica-set count).

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `replicaid` | text | Application identifier. Join key to `cqx_data.replica_set`. |
| `is_focus_db` | boolean | Focus scope flag. See [[concepts/focus-db]]. |
| `complexity_category` | text | Design-complexity label: Very High / High / Medium / Low / Very Low / Not Analyzed. |
| `complexity_category_simplified` | text | Collapsed family: Very High / Medium / High / Low / Very Low / Not Analyzed. |
| `complexity_index` | integer | Ordinal over the simplified family: 0 = Not Analyzed, 1 = Low / Very Low, 2 = Medium / High, 3 = Very High. Sort key for "most complex". |
| `usage_category` / `usage_category_simplified` | text | Usage tier label. See [[concepts/usage-tier]]. |
| `usage_index` | integer | Ordinal usage grade. |
| `remediation_classification` | text | Application triage label. See [[concepts/remediation-classification]]. |
| `remediation_index` | integer | Ordinal for the remediation classification. |
| `lifecycle_vector` | text | ETL-computed lifecycle bucket (same enum as `cqx_data.replica_set`). This view is focus-only, so it is never NULL here. See [[concepts/lifecycle-vector]]. |
| `template_inherits_from` / `template_acts_as_master` | text[] | Template association arrays. |
| `last_accessed` | date | Most recent access. |
| `users_active` / `user_access_days` | integer | Headline usage counts. |

## Relationships

- Join to `cqx_data.replica_set` on `replicaid` for the full application row.
- `db_type_id` -> `cqx_data.database_inventory.db_type_id`.

## Typical use

Rank analysed applications by complexity grade:
```sql
SELECT replicaid, title, complexity_category, complexity_index
FROM cqx_data.replica_set_summary
WHERE is_focus_db AND complexity_index > 0
ORDER BY complexity_index DESC
```

## Notes and caveats

- The `*_index` columns exist so you can sort by an ordinal grade without parsing labels. `complexity_index = 0` means **Not Analyzed**, not "least complex" — exclude it (`> 0`) when ranking by complexity.
- This is a categorical rollup. For the numeric complexity score, per-language code volumes, or template-similarity columns, use [[schema/tables/replica-set]] and [[schema/tables/design-summary-by-replica-set]].
- The label families here match the dashboards; the same "Not Analyzed" caveat applies — most of the estate is unanalysed because only Focus DBs and templates are scanned. See [[concepts/database-complexity]].

## Sources

CQX schema reference; label/index mapping verified on the POC instance 2026-06-15.
