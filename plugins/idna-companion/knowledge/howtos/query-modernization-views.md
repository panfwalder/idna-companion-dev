---
id: howtos/query-modernization-views
title: "How to query the modernization views"
type: howto
status: verified
tags: [domain/idna, topic/modernization, topic/64bit, topic/nomad]
related:
  - schema/tables/modernization-vector
  - schema/tables/modernization-plan
  - schema/tables/modernization-worklist
  - concepts/about-modernization-vector
  - concepts/about-modernization-plan
  - schema/tables/design-insight-overview-byrepset
  - schema/tables/code-source
  - schema/tables/template-ancestry
  - schema/tables/modernization-lineage-dossier
  - schema/tables/modernization-session-steps
requires_capability: none
source: modernization-vector-integration plan (verified queries, ifa test instance 2026-07-09/23); template-lineage views and the revised reach identity (ETL ADRs 0002-0004) verified live 2026-09-24; the n_hcl_current identity term (ETL ADR 0006) verified live 2026-09-30
last_reviewed: 2026-10-01
---

# How to query the modernization views

Query mechanics only — for what the values *mean*, read [[concepts/about-modernization-vector]] and [[concepts/about-modernization-plan]] first.

**Tools first:** each question shape below has a dedicated MCP tool that works in EVERY session profile (restricted sessions have no `execute_sql`). The raw SQL is the privileged/custom-slice path — see [[howtos/use-source-code-remediation-mcp-tools]] for the tool routing map.

## Which view answers which question

| Question shape | Tool (any profile) | View (privileged SQL) |
|---|---|---|
| "Is app X ready for target Y? What kind of fix?" | `modernization_for_app` | `modernization_vector` (per-app row) |
| "Which apps aren't ready?" / readiness split + KPI | `modernization_plan_overview` | `modernization_vector` grouped by vector value |
| "Where do we start? What pays off most?" | `modernization_plan_overview` | `modernization_plan` ordered by `item_rank` |
| "Which blocks do we fix once and reapply?" | `modernization_plan_overview` (top blocks) | `modernization_worklist` ordered by `item_rank` |
| "What exactly do I do in session X?" | `modernization_session_dossier` (work rows) | `modernization_worklist_by_session` filtered by (`session_kind`, `session_anchor`) |
| "In what order do I do it?" (the procedure) | `modernization_session_dossier` (`steps`) | `modernization_session_steps` by the session key, `ORDER BY step_rank` |
| "Which standard template was this master forked from?" | `modernization_lineage_overview` (the `master` header) | `template_ancestry` |
| "What did we customize? What survives the update?" | `modernization_lineage_overview` (counts), then `modernization_lineage_dossier` (drill-down) | `modernization_lineage_dossier` (element rows / finding rows) |
| Quick per-app headline (built-in targets only) | `modernization_for_app` | `replica_set.modernization_vector_notes64bit` / `..._nomad` |

The session steps (ETL ADR 0004) come through `modernization_session_dossier` (`steps`; on an older MCP build without that field, query the view in privileged sessions). The two lineage views (`template_ancestry`, `modernization_lineage_dossier`; ETL ADRs 0002–0003, 0005) come through `modernization_lineage_overview` (counts) and `modernization_lineage_dossier` (the drill-down) in any profile; privileged `execute_sql` stays the custom-slice fallback. On an older MCP build without those tools, `execute_sql` is the only path; restricted sessions then answer from [[concepts/about-template-lineage]] and say that lineage detail is not available in this session.

## Rule 1: ALWAYS filter by ruleset

The vector, plan, worklist, dossier-by-session and steps views are per-target. So are the lineage dossier's finding rows; its element rows and `template_ancestry` are ruleset-independent. `modernization_vector` has one row per (app × ruleset) — unfiltered counts double (574 = 287 × 2 on a reference instance). Built-in ids: Notes 64bit `71f16a9a-8fc6-4dfe-add8-3a43c539d353`, Nomad `a8efac35-3e54-48f4-aa27-4c342529ed0a` (resolve custom ones via `ruleset_name`).

## Rule 2: category selection uses `modernization_search`, not the raw vector

`modernization_search` is the vector padded to full depth with a trailing slash — built for prefix matching:

```sql
-- everything under remediate_master, incl. all qualifier variants
WHERE modernization_search LIKE 'template/remediate_master/%'
```

The trailing slash prevents `design_family_#8` from matching `#80`. The raw `modernization_vector` is unpadded (a bare `template/remediate_master` means "fully covered") — string-match it only for exact values.

**Ready = prefix `ready/`.** `ready/with_notes` apps are ready (ETL ADR 0003), so the readiness split is `modernization_search LIKE 'ready/%'` vs. the rest — NOT `modernization_vector = 'ready'`, which drops them:

```sql
SELECT (modernization_search LIKE 'ready/%') AS is_ready, count(*) AS apps
FROM cqx_data.modernization_vector
WHERE ruleset_id = '<ruleset_id>'
GROUP BY 1
```

## Rule 3: query `items` defensively

`items` has **one entry per finding** (a block can appear under several insights) and can be very large on template-derived apps. Project fields, dedupe by `code_hash` for block-level analysis (`n_fix_items` counts distinct blocks), check `jsonb_array_length(items)` first, use `LIMIT`:

On TRIAL licenses check `detail_masked` first (2026-08 ETL revision): masked rows carry ordinal aliases instead of document/element names and null `noteid`/`designer_link` — `code_hash*`/`clearance` stay usable for block-level analysis. Present masked rows as "detail withheld, a full license unlocks it", never "no work". Master anchors across all four views are per-run `Master n` aliases (join-consistent within a run). See [[schema/topics/trial-license-masking]].

```sql
SELECT DISTINCT item ->> 'code_hash' AS code_hash, item ->> 'clearance' AS clearance
FROM cqx_data.modernization_vector mv, jsonb_array_elements(mv.items) AS item
WHERE mv.replicaid = '<replicaid>' AND mv.ruleset_id = '<ruleset_id>'
```

## Rule 4: three sources can locate a code block — only one reconciles with the vector

- **`design_insight_overview_byrepset`** (with a `ruleset_id` predicate) — findings at focus-repset grain; this is the vector's reach source *by construction*: `count(DISTINCT replicaid)` per `source_code_hash` reconciles 1:1 with `n_shared_items`/`n_solo_items` — **with one caveat**: the modernization precompute additionally applies the `etl_modernization_ignore_codeblocks` FP-suppression config, which this view does not. Raw-finding reach reconciles exactly when no suppressions are configured; with suppressions it counts dismissed occurrences too — **and the gap is itself reconcilable** since the vector documents every dismissal (`n_dismissed_items`/`dismissed_items`; recipe below).
- **`find_code_block_usage` (MCP tool)** — the same join, input `code_hash_id`, restricted-profile approved. Pass `target`/`ruleset_id` for grouped per-app rows whose `row_count` IS the per-ruleset focus reach; without it the rows are occurrence-grain across all rulesets (row_count is NOT app reach). Same suppression caveat as above. Grouped rows carry NO design-element context (which form/library/agent holds the block) — when locations matter, pair the grouped call with occurrence mode or `get_code_block`; never present a bare app list as a block's spread.
- **`code_element` + `code_source`** — raw presence of a hash anywhere in the licensed estate, flagged or not, at **database-instance grain**, with design location (`named_path`) and source text. Answers "show me the code / where exactly does it sit" — **never** use it to reconcile shared/solo (no ruleset scoping, instance grain, presence ≠ finding). Large surface (~9M rows): always filter by hash.

## Rule 4b: dismissed non-issues reconcile the suppression gap

Since the 2026-07 revision, config-dismissed findings are documented on the
vector instead of vanishing (see [[concepts/about-modernization-vector]]):
`dismissed_items` (same fields as `items` minus `clearance`, plus
`assessment: 'non_issue'`) and `n_dismissed_items`. The identity per
(app × ruleset):

```
raw finding occurrences = jsonb_array_length(items) + n_dismissed_items
```

```sql
-- Which apps carry dismissals for this target, and what was dismissed?
SELECT replicaid, title, n_dismissed_items,
       jsonb_array_length(items) AS active_entries,
       d ->> 'finding_name' AS dismissed_finding,
       d ->> 'code_hash'    AS dismissed_hash
FROM cqx_data.modernization_vector mv, jsonb_array_elements(mv.dismissed_items) AS d
WHERE mv.ruleset_id = '<ruleset_id>' AND mv.n_dismissed_items > 0
```

Grain warning: `n_dismissed_items` counts occurrences (like `items` entries),
NOT blocks — never compare it with the block-grain `n_fix_items`. Availability:
the columns exist from the 2026-07 ETL release onward.

## Rule 5: block lists show `scope` next to `reach`

When presenting worklist blocks, always include `scope` alongside `reach` — they answer different questions (reach = how many kept apps carry the block; scope = whether fixing it spans one session home or several), and omitting scope invites the T9 conflation. A reach-3 block can be `scope = 'session'` when its carriers form one unify session.

## Rule 6: reach is usually a lookup now

`modernization_worklist.reach` (and the identity `reach = n_cleared_by_refresh + n_persists_after_update + n_hcl_current + |apply| + |review| + |scan_first|`, ETL ADR 0003 as amended by ETL ADR 0006 — covered occurrences whose block survives the template update count as persists; occurrences of current HCL code outside lineage master coverage count as `n_hcl_current`, never as work) materializes per-block reach — prefer it over recomputing. Recompute from `design_insight_overview_byrepset` only when reconciling doubted numbers or analyzing blocks outside the worklist.

## Traps in query results (short list)

- **Sunset apps have no vector rows** but DO count in vector shared/solo reach and appear in `design_family` arrays. Worklist `reach` conversely counts **kept apps only** — two different scopes.
- **shared ≠ family-shared:** `n_shared_items` is estate-wide exact-hash reach. Family siblings drift → different hashes → a member can be 8/0 next to a 4/4 sibling. Reconcile per block hash before calling numbers weird.
- **Coverage NULLs mean unscanned master** (unknown, not zero) — the action is scanning the master.
- **On lineage routes, covered ≠ cleared.** For `upgrade_standard` / `rebase_ancestor` use the vector's `n_cleared_after_update` and `n_hcl_notes` (and the display), not `n_covered_by_master`.
- **Session steps have no fixed scan order** — always `ORDER BY step_rank`.
- **Dossier joins use the pair** (`session_kind`, `session_anchor`), never anchor text alone.
- **`items` holds ACTIVE findings only** — config-dismissed findings live in `dismissed_items`. A raw-source count that exceeds the `items` count is usually dismissals, not a defect (Rule 4b).
