---
id: schema/tables/modernization-plan
title: cqx_data.modernization_plan
type: schema
status: verified
tags: [domain/idna, topic/modernization, topic/64bit, topic/nomad]
related:
  - concepts/about-modernization-plan
  - concepts/about-modernization-vector
  - schema/tables/modernization-vector
  - schema/tables/modernization-worklist
  - howtos/query-modernization-views
requires_capability: none
source: ETL cqx_data_ifa/010_cqx_data.initialize_components_structure.pxsql (view) over dda_dm.modernization_plan (ETL script 286); types verified against live instance 2026-07-23
last_reviewed: 2026-07-23
---

# cqx_data.modernization_plan

## Purpose

The ordered **session queue** — the customer-facing modernization work plan ([[concepts/about-modernization-plan]]). One row per session per ruleset: a session is one human sitting on one design container (master template, design family, or standalone app). **Sessions only** — block-grain piece-work lives in [[schema/tables/modernization-worklist]] (rows with `item_kind = 'patch'` existed here historically, until 2026-07-11).

## Key columns

| Column | Type | Meaning |
|---|---|---|
| `ruleset_id`, `ruleset_name` | text | The compatibility target. One plan per target — **filter on it.** |
| `item_kind` | text | Always `session` (historical column; patches moved to the worklist). |
| `session_kind` | text | Mirrors the vector route: `template/remediate_master`, `template/upgrade_standard`, `template/unify_similar`, `template/investigate_inheritance/*`, `direct/remediate_code`. |
| `anchor` | text | The container: master template name, design-family id, or the app itself. NOTE: this view names the column `anchor` — the vector and the dossier view call the same value `session_anchor`; don't guess the name across views. |
| `item_display` | text | ETL-assembled human sentence ("Remediate master pana_StdR9Mail", "Upgrade standard master StdR9TeamRoom to the current HCL version") — quote it. |
| `item_rank` | integer | Queue position: apps cleared → apps → usage cleared. |
| `replicaids` | text[] | Session membership (the apps this session concerns). |
| `n_apps` | integer | Membership count. |
| `n_apps_green` | integer | **THE session KPI: apps this one session ALONE turns fully green.** Ranking driver. |
| `n_apps_partial` | integer | Apps the session helps but does not clear (surplus/diverging work remains). |
| `uad_90d_cleared`, `uad_365d_cleared` | bigint | Usage-weighted companions ("how much usage goes green") — tie-breakers, not the primary rank. |
| `n_app_fix_items` | integer | App-side blocks involved. |
| `n_master_blocks_to_fix` | integer | The master's own finding blocks (the real fix locations behind covered items) — itemized per block in the worklist. |
| `master_unscanned` | boolean | Master design not collected: session starts with a scan, not a fix. |

## Typical use

Top sessions for a target (the "where do we start" answer):
```sql
SELECT item_rank, session_kind, anchor, item_display, n_apps, n_apps_green
FROM cqx_data.modernization_plan
WHERE ruleset_id = '<ruleset_id>'
ORDER BY item_rank
LIMIT 10
```

A session's worksheet: drill into `modernization_worklist_by_session` by (`session_kind`, `anchor`) — see [[schema/tables/modernization-worklist]].

## Notes and caveats

- **Rank by `n_apps_green`, not `n_apps`** — membership includes apps the session only partially helps.
- Distinguish clearly in answers: green = fully cleared by this session alone; partial = helped, residual work remains.
- Unify/investigate sessions **assume human decisions resolve** (reconcilability judgment, possible cohort splits); present their `n_apps` as potential, and partial outcomes (two masters instead of one; "not reconcilable") as legitimate exits.
- Session displays may reference sunset design-family peers ("+N sunset peer(s)") — evidence for the unification decision, not remediation targets.
- Advisory framing: the ranking has a stated rationale (apps cleared); sequencing decisions (staffing, politics) remain the customer's.
