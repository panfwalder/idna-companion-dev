---
id: schema/tables/code-block-hcl-origin
title: cqx_data.code_block_hcl_origin
type: schema
status: verified
tags: [domain/idna, topic/modernization, topic/64bit, topic/nomad]
related:
  - schema/tables/modernization-worklist
  - schema/tables/modernization-vector
  - schema/tables/modernization-lineage-dossier
  - concepts/about-modernization-vector
  - concepts/about-template-lineage
requires_capability: none
source: ETL cqx_data_ifa/010_cqx_data.initialize_components_structure.pxsql (view + COMMENT) over dda_dm.code_block_hcl_origin (ETL script 224); ETL ADRs 0006 and 0007; column types and row counts verified against the live instance 2026-10-01; Companion plan code-block-eval-fixes Part E (G6, 2026-10-02); HCL Domino Designer 14 help: "Refreshing a design", "Preventing design changes"
last_reviewed: 2026-10-02
---

# cqx_data.code_block_hcl_origin

## Purpose

**The HCL origin of a code block** (ETL ADRs 0006 / 0007): one row per code hash that ships in any HCL standard template of the shipped reference catalog. It answers "is this exact code HCL template code, from which template family and which Notes versions, and is it still in the current HCL version?" — for every block, including blocks without a worklist row (e.g. a block whose every occurrence sits in `ready/with_notes` apps).

Current HCL code is HCL's to own: no code fix to author, a note rather than work (see [[concepts/about-modernization-vector]]). Older-only code is a caution for the developer, not a note.

## When current HCL code fails on the target

"No code fix to author" means the fix is not the customer's to write. It does not mean the code works on the target: the finding remains visible and still needs an execution-context review (findings measure non-portable code, not guaranteed breakage).

- **Does an update or refresh clear it?** For a covered `persists_hcl_owned` occurrence, the lineage verdict says the code remains after the current cataloged template update. For a verbatim copy, use `hcl_origin` to identify the family that still carries the exact code; reconnecting or refreshing from that same current family does not remove it. Do not generalize `in_current` to every possible template family or future HCL release: a block is current when the newest member of any live family carries it.
- **What to advise.** First establish whether the affected code path executes in the target environment and in what context (Notes client, Nomad, or Domino server). If an agent is unused, disabling it is valid containment; if an element is already disabled or its library code is unreachable, record that containment rather than implying that every element can be disabled. If the path is needed and executes on the target, test it in that execution context. If the compatibility issue is confirmed, raise it with HCL Support and name the template family and versions from `hcl_origin`. These are review and containment steps, not author/apply work, and they do not remove the static finding.
- **No hand-edit as the remediation.** Do not plan a hand-edit of current HCL code as the remediation. Editing it creates a customer customization that must be maintained and retested. In a lineage app, a later refresh replaces that edit only when the individual design element remains linked to the template and is not protected from design refresh or replace.
- **When HCL ships a correction.** When HCL ships a corrected version of the relevant template family, an unprotected occurrence that still inherits that design element receives the change through the appropriate template update and refresh. A verbatim copy, an unlinked element, or a protected element receives no automatic upstream change; it needs an explicit, governed update.

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `code_hash` | text | The universal block hash that `modernization_vector` items and `modernization_worklist` carry. **The join key.** |
| `hcl_families` | text[] | Every catalog family the hash ships in, retired families included. |
| `first_version`, `last_version` | numeric | The span of shipped Notes versions that carry the hash. |
| `live_families` | text[] | Live families whose newest shipped member contains the hash. |
| `in_current` | boolean | TRUE = current HCL code: in the newest member of at least one live family (HCL's to own, ETL ADR 0006). |
| `current_version` | numeric | The newest such version. |
| `hcl_origin` | text | The name-free origin sentence (family labels and Notes versions only), the same sentence the worklist carries. Current: "HCL Mail template code (Notes 8 - 14.5; in the current shipped catalog version 14.5)". Older-only: "Older HCL Mail template code (Notes 7 - 8.5; changed or removed by 14.5) - compare with the current HCL version before hand-fixing". |
| `family_ranges` | jsonb | Per family, an object with exactly the keys `family`, `first_version`, `last_version`, `current_version`, `is_live`, `in_current`; current-live families first. |

## Relationships

- **Join by `code_hash`, never by `code_hash_id`.** `code_hash_id` is environment-local and is not a column of this view.
- `modernization_vector.items[].code_hash` → `code_hash`: the origin of each finding block of an app ([[schema/tables/modernization-vector]]).
- `modernization_worklist.code_hash` → `code_hash`: the worklist already carries `hcl_origin` and `hcl_in_current` per block ([[schema/tables/modernization-worklist]]); use this view for blocks without a worklist row.
- Where a master's family is unmapped, the lineage verdict `persists_hcl_owned` in [[schema/tables/modernization-lineage-dossier]] rests on the same shipped-catalog fact.

## Typical use

```sql
-- The HCL origin of one block
SELECT in_current, hcl_origin, family_ranges
FROM cqx_data.code_block_hcl_origin
WHERE code_hash = '<code_hash>'
```

## Notes and caveats

- **"Current" means the product's reference catalog, not the customer's installed release.** Name the catalog version (`current_version`, e.g. "current shipped catalog version 14.5"); never say "your current version".
- **Older-only rows** (`in_current = false`) are code HCL has since changed or removed. They are not `hcl_owned`; where such a block is work, the developer compares it with the current HCL version before hand-fixing.
- **A block absent from this view** is not shipped HCL code: a NULL origin after a LEFT JOIN means "not in the shipped HCL catalog", not "unknown".
- **Name-free and identical on every licence.** No template names, no person names, no aliasing: trial and full licences show the same rows. The view rebuilds `family_ranges` from its six keys, so no producer-side field can leak a name.
- Size: 14,630 rows, 11,261 current, on a reference instance (2026-10-01). The longest `hcl_origin` is 730 characters; `modernization_for_app` and `modernization_session_dossier` shorten it to 256 characters and say so (`origin_shortened`) — the full sentence is here, by `code_hash`.
- Precomputed by ETL (script 224).
