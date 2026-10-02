---
id: schema/tables/modernization-lineage-dossier
title: cqx_data.modernization_lineage_dossier
type: schema
status: verified
tags: [domain/idna, topic/modernization, topic/64bit, topic/nomad]
related:
  - schema/tables/template-ancestry
  - schema/tables/modernization-worklist
  - schema/tables/modernization-session-steps
  - schema/tables/modernization-vector
  - schema/tables/code-source
  - schema/topics/trial-license-masking
  - concepts/about-modernization-vector
  - schema/tables/code-block-hcl-origin
requires_capability: none
source: ETL cqx_data_ifa/010 (view + COMMENT) over dda_dm.modernization_lineage_dossier (create 030, ETL script 226); ETL ADRs 0002, 0003 and 0005 (design-document grain); column list and types verified against the live instance 2026-09-25; the persists_hcl_owned refinement from the shipped catalog (ETL ADR 0006) checked 2026-10-01; element pair = the code inside the document per ETL ADR 0008 (2026-10-01)
last_reviewed: 2026-10-02
---

# cqx_data.modernization_lineage_dossier

## Purpose

**The lineage worksheet for every master in [[schema/tables/template-ancestry]].** It answers the two questions an admin has before touching a forked or standard master:

- **"What did we change back then?"** — `element` rows classify the master's design documents against its standard ancestor.
- **"What survives the update?"** — `finding` rows give every fix-item block a fate verdict.

The dossier feeds rebase and upgrade sessions, the `ready/with_notes` state, and the verdict-aware worklist (ETL ADRs 0002 and 0003). Renamed in place from `modernization_rebase_dossier` on 2026-09-23 because it serves more than rebase sessions.

## Grain (ETL ADR 0005, 2026-09-25)

Both row kinds sit at one **design-document grain**:

- **`element` rows:** one per code-carrying design document of the master (a document with at least one hashed code block and a resolvable element identity; documents without code have no row). Same-named documents are classified each on its own.
- **`finding` rows:** one per (`ruleset_id`, code block, design document). A block that sits in three documents has three rows. Its `verdict`, `block_in_ancestor` and `block_in_latest` are block-level and repeat on each of them.
- **Count verdicts per block with `count(DISTINCT code_hash_id)`**, never `count(*)`, which counts block × document rows.
- **Join the row kinds per document** on `tmpl_name` plus `ddoc_id` (full license) or `design_document_type` + `design_document_name` (trial, where `ddoc_id` is NULL). The join example below combines both in one predicate that works on either license.
- A finding row whose document could not be resolved keeps its row with `ddoc_id` NULL (fail-soft). On a full license it still carries its own document type and name; on trial its `design_document_name` is NULL (no alias without `ddoc_id`). It matches no element row on either license.

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `tmpl_name` | text | The master. |
| `row_kind` | text | `element` or `finding`. |
| `classifier_mode` | text | `A` = estate-scanned ancestor (element identity on both sides); `B` = shipped reference (hash grain only); `F` = standard-master fate rows (finding rows only, judged against the family's newest version or, for an unmapped family, against the shipped catalog). |
| `ddoc_id` | bigint | The design document's internal id in the master's scan — the join key between the row kinds. It changes on rescan: use it to join within one query, never present it as an identifier or compare it across runs. NULL on trial-masked rows and on unresolved finding rows. |
| `design_document_type` | text | The document's type as one token (`Form`, `View`, `Subform`, `Scriptlibrary`, `Sharedfield`, …), both row kinds. |
| `delta_class` | text | Element rows: `inherited` / `modified` / `added`. Finding rows: the class of the row's own document; the verdict stays block-level, so a block that also sits in an added document reads `reapplied_with_element` even on its row in an inherited document. NULL in mode F and when no element row matches (document or element identity unknown). |
| `n_hashes`, `n_custom_hashes` | integer | Element rows: code hashes in the document; hashes not in the ancestor. |
| `n_removed_ancestor_hashes` | integer | Element rows, mode A only: ancestor code the fork removed (NULL in mode B). |
| `ruleset_id`, `ruleset_name` | text | Finding rows only — the compatibility target. **Filter on it.** Element rows are ruleset-independent (NULL). |
| `code_hash_id`, `code_hash` | bigint, text | The block (joins to [[schema/tables/code-source]]). |
| `design_document_name` | text | The document's name, both row kinds (one per-document alias on trial; NULL on an unresolved trial finding row). |
| `finding_name`, `code_element_name` | text | Finding rows: what was flagged and in which code elements — `'; '`-separated lists within this one document only. `code_element_name` is NULL on element rows. The names are element detail inside the document, never its location: since ETL ADR 0008 an opaque DXL identifier for `Code` (an event, LotusScript section, entry sub, sub / function name, or raw `code[1]`), otherwise the host element's name (`button[1]`, a field name); older builds may repeat the document name or show `globals[1]`. |
| `block_in_ancestor`, `block_in_latest` | boolean | Is the block in the ancestor / in the family's newest version? `block_in_latest` NULL = newest version unresolved. |
| `verdict` | text | Finding rows — see below. |
| `detail_masked` | boolean | Trial only — identity on this row is withheld, not absent. |

## Finding verdicts

| Verdict | Meaning | Worklist `work_kind` |
|---|---|---|
| `custom_fix_required` | Custom code — not in the ancestor. It survives any upgrade; the developer owns the fix. | `author` / `apply` (real work) |
| `reapplied_with_element` | Inherited code carried inside an element the fork added (or whose identity is unknown — conservative): it comes back when that element is re-applied. Verify after the update. | `verify_update` |
| `cleared_by_rebase` | Fork rows: inherited code the family's newest version no longer contains — the rebase removes it. | `cleared_by_update` |
| `cleared_by_upgrade` | Standard-master rows: code the family's newest version no longer contains — the upgrade removes it. | `cleared_by_update` |
| `persists_hcl_owned` | Inherited code still present in the current HCL version (or the master already is the newest version; or, where the family is unmapped, the shipped HCL catalog has the exact code in its current version — `block_in_latest` stays NULL then, ETL ADR 0006) — HCL's to maintain, no code fix to author. | `hcl_owned` |
| `inherited_latest_unknown` | The template family cannot be mapped, or its newest version is unresolved — fate unknown, unless the shipped HCL catalog shows the block's exact code in the current version (then `persists_hcl_owned`, ETL ADR 0006). Verify after the update. | `verify_update` |

An app on a lineage route whose active fix items are ALL `persists_hcl_owned` is `ready/with_notes` — ready, with notes on HCL-shipped code (see [[concepts/about-modernization-vector]]). The same state also covers a non-lineage app with no dossier rows: all its active items are current HCL template code and none is a diverging copy of such code (a diverging copy is review work; ETL ADR 0007 D-6). Its notes are verbatim copies, not lineage verdicts.

## Relationships

- `tmpl_name` joins [[schema/tables/template-ancestry]]. Verdicts surface per session home in `modernization_worklist.sessions[].block_verdict` and as the `work_kind` of [[schema/tables/modernization-worklist]]'s by-session view.
- Step counts in [[schema/tables/modernization-session-steps]] (re-apply, verify carryover) come from the element rows, so they count design documents.

## Typical use

```sql
-- What did we customize? (document deltas of one master)
SELECT delta_class, count(*) AS documents, sum(n_custom_hashes) AS custom_hashes
FROM cqx_data.modernization_lineage_dossier
WHERE tmpl_name = '<master>' AND row_kind = 'element'
GROUP BY delta_class

-- What survives the update? (finding verdicts for one target, per block)
SELECT verdict, count(DISTINCT code_hash_id) AS blocks
FROM cqx_data.modernization_lineage_dossier
WHERE tmpl_name = '<master>' AND row_kind = 'finding' AND ruleset_id = '<ruleset_id>'
GROUP BY verdict

-- Which findings does each modified document carry? (one join for both licenses)
SELECT e.design_document_type, e.design_document_name, f.code_hash_id, f.verdict, f.finding_name
FROM cqx_data.modernization_lineage_dossier e
JOIN cqx_data.modernization_lineage_dossier f
  ON f.tmpl_name = e.tmpl_name
 AND COALESCE(f.ddoc_id, -1) = COALESCE(e.ddoc_id, -1)
 AND f.design_document_type = e.design_document_type
 AND f.design_document_name = e.design_document_name
 AND f.row_kind = 'finding' AND f.ruleset_id = '<ruleset_id>'
WHERE e.tmpl_name = '<master>' AND e.row_kind = 'element' AND e.delta_class = 'modified'
ORDER BY e.design_document_type, e.design_document_name, f.verdict
```

This join works on both licenses. On a full license `ddoc_id` decides, so same-named documents stay apart; on trial both `ddoc_id` are NULL (the `COALESCE` makes them equal) and the per-document alias decides. An unresolved finding row (`ddoc_id` NULL on a full license, name NULL on trial) matches nothing. Type and name are identical on both row kinds of one document by construction.

## Notes and caveats

- **Mode B cannot see everything.** Against a shipped reference (hash grain only) the dossier cannot tell a fully rewritten element from a new one, and cannot see deletion-only customizations at element grain. Say "all-custom code elements (possibly new or fully rewritten)", never "fork-only elements", for mode B counts.
- **Blocks with unknown element identity classify conservatively.** Blocks with NULL code hash are excluded.
- **Findings in untouched standard code are not the customer's work.** They die with the upgrade (`cleared_by_*`) or stay with HCL (`persists_hcl_owned`). Never present them as fixes to author. The template family and versions a block ships in: [[schema/tables/code-block-hcl-origin]], by `code_hash`.
- TRIAL licenses: master names appear as per-run `Master n` aliases. Unless all of the master's instances are trial-selected (practically always masked): `design_document_name` becomes one ordinal alias per document (`Form 7`, numbered within master and type), the same on both row kinds and valid for one snapshot only (NULL on a finding row whose document could not be resolved); `ddoc_id` and `code_element_name` are NULL; `design_document_type` stays visible. `detail_masked = true` means "detail withheld", never "no work" — counts and verdicts stay honest. See [[schema/topics/trial-license-masking]].
- Access: granted to the restricted role (ADR 0006 appendix). Two MCP tools read this view in any profile: `modernization_lineage_overview` (counts only) and `modernization_lineage_dossier` (the filtered drill-down: documents with their findings nested). On an older MCP build without them, privileged sessions reach the view via `execute_sql`.
