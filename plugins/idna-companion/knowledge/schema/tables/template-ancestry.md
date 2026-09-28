---
id: schema/tables/template-ancestry
title: cqx_data.template_ancestry
type: schema
status: verified
tags: [domain/idna, topic/modernization, topic/64bit, topic/nomad]
related:
  - schema/tables/modernization-lineage-dossier
  - schema/tables/modernization-vector
  - schema/tables/modernization-plan
  - schema/topics/similarity
  - schema/topics/template-inheritance
  - schema/topics/trial-license-masking
  - concepts/about-modernization-vector
requires_capability: none
source: ETL cqx_data_ifa/010 (view + COMMENT) over dda_dm.template_ancestry (create 029, ETL script 225); ETL ADRs 0002 and 0003; column types verified against the live instance 2026-09-24
last_reviewed: 2026-09-25
---

# cqx_data.template_ancestry

## Purpose

**Template lineage per estate master template** — one row per master name. It answers "which standard template is this master really derived from, and how far has it drifted?", regardless of findings: the template-hygiene lens (ETL ADR 0002).

A very common Notes pattern: someone copies a standard template (the copy severs inheritance), changes a few things, and the result becomes "their own" master — a detached fork. This view recovers the lost ancestry from design similarity. It drives the `template/rebase_ancestor` and `template/upgrade_standard` routes and feeds [[schema/tables/modernization-lineage-dossier]].

Two row kinds:

- **Fork rows** (`is_standard_master = false`): the master carries its own name; its design links to a STANDARD identity. The row names that ancestor, the similarity, and the family's newest member.
- **Standard-master rows** (`is_standard_master = true`): the master already carries a standard name. Fate context only — own family and newest family member; `ancestor_*` columns are NULL (ETL ADR 0003).

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `tmpl_name` | text | The estate master template name. |
| `is_standard_master` | boolean | TRUE = standard-named master (fate row). |
| `ancestor_name` | text | Best standard ancestor identity. NULL on standard-master rows. |
| `ancestor_kind` | text | `standard_estate` — an estate template operated under a standard-recognized name (the actionable anchor); `standard_shipped` — a shipped standard-template reference row only. |
| `ancestor_similarity` | numeric | Design similarity to the ancestor; 100 = identical hash set. |
| `is_rename_only` | boolean | `ancestor_similarity = 100` — identical code base under another name ("upgrade directly"). |
| `family_name` | text | Catalog template family (e.g. the mail family); NULL = unmapped. |
| `latest_name`, `latest_notes_version` | text, numeric | Newest known member of the family — the upgrade reference. NULL when unresolved. |
| `latest_source` | text | `estate_scanned` or `shipped`. |
| `n_distinct_sets` | integer | Distinct design hash sets across the master's instances — replica drift. |

## Relationships

- `tmpl_name` = the `anchor` of `template/rebase_ancestor` / `template/upgrade_standard` sessions in [[schema/tables/modernization-plan]] and the `ancestor_name` / `ancestor_kind` / `ancestor_similarity` columns of [[schema/tables/modernization-vector]].
- `tmpl_name` = `modernization_lineage_dossier.tmpl_name` (one dossier per ancestry row).

## Typical use

```sql
-- Detached forks of standard templates, closest first
SELECT tmpl_name, ancestor_name, ancestor_kind, round(ancestor_similarity) AS pct,
       is_rename_only, family_name, latest_name, latest_notes_version
FROM cqx_data.template_ancestry
WHERE NOT is_standard_master
ORDER BY ancestor_similarity DESC
```

## Notes and caveats

- **Pick rule:** the most similar candidate wins across estate and shipped standards; an estate-operated standard breaks ties (ETL ADR 0003 clause 4, superseding the ADR 0002 estate-first class preference). Only links above the configured design-cluster threshold count.
- **Family, not similarity, names the upgrade target.** `latest_name` comes from family membership (newest estate-scanned member, else newest shipped reference), never from similarity links. The recommended action stays target-agnostic: upgrade to the current version of the standard template, then re-apply the customizations.
- **Lineage regardless of findings.** An ancestry row does not mean work exists; routing decides that from the findings.
- **Non-standard estate-master ancestors are not modeled** (ETL v2 item) — ancestry here always points at a standard identity.
- TRIAL licenses: `tmpl_name`, `ancestor_name` and `latest_name` appear as per-run `Master n` aliases (shipped standard names included); `family_name` and versions stay clear. See [[schema/topics/trial-license-masking]].
- Access: granted to the restricted role (ADR 0006 appendix). `modernization_lineage_overview` returns one master's ancestry row as its header, in any profile; on an older MCP build without that tool, privileged sessions reach the view via `execute_sql`.
