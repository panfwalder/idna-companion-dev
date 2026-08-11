---
id: howtos/use-application-lifecycle-mcp-tools
title: Use the application-lifecycle MCP tools
type: howto
status: verified
tags: [domain/idna, topic/consolidation, topic/modernization, audience/consultant]
related:
  - playbooks/application-lifecycle/lifecycle-routing
  - playbooks/application-lifecycle/sunset-funnel
  - playbooks/application-lifecycle/keep-funnel
  - playbooks/application-lifecycle/stakeholder-cohort
  - concepts/usage-tier
  - concepts/remediation-classification
  - concepts/compatibility-ruleset-routing
  - concepts/document-age-timestamps
requires_capability: none
source: app-lifecycle-playbook-rewrite cycle 2; pseudo-interfaces contract
last_reviewed: 2026-07-04
---

# Use the application-lifecycle MCP tools

## Question

"Which MCP tool should I use for this lifecycle / consolidation question, and how do they chain?"

## Approach

Lifecycle work runs on the semantic lifecycle MCP tools. The **base** lifecycle bucket is now an ETL-computed column (`cqx_data.replica_set.lifecycle_vector`, see [[concepts/lifecycle-vector]]) that the tools read — and that serves as a degraded SQL fallback if the MCP is down (base bucket only; no action patterns, scan-freshness, or per-app dispositions). The playbooks remain the "why" and "when" layer; the tools are the execution layer. All eight tools are restricted-mode-safe (`access_profile: 'restricted'`).

## Required inputs

- Session preflight via `get_idna_session` (data-currency, `source_code_access_available`, `dashboard_base_url`).
- For keep-side tools, a forward `target` mapped from customer intent: `notes-64bit` | `nomad-web-mobile` | `multiple` | `none-yet` (see [[concepts/compatibility-ruleset-routing]]).
- A `replicaid` for the per-app detail tools.

## Routing map

- "Which apps can we retire?" / "How big is the consolidation problem?" / "Where do we start?" → `lifecycle_exposure_summary` (per-disposition histogram + context).
- "Show me the sunset candidates." → `sunset_candidates`.
- "Tell me about this sunset app." → `sunset_disposition`.
- "Which instance do we archive?" → `archival_instance_pick` (only for a `sunset/archive` app with >1 instance).
- "Show me the keep / modernization queue." → `keep_candidates` (needs `target`).
- "Tell me about this keep app's disposition." → `keep_disposition`.
- "Diagnose this quiet keep app." → `qualify_app_analysis`.
- "Who depends on this app? / which departments use it?" → `stakeholder_cohort` (also the cohort surface for `sunset/archive` context; the keep/qualify path reads the same cohort from `qualify_app_analysis.layers.stakeholder_cohort`).

## Chaining (typical consolidation walkthrough)

1. `get_idna_session` — preflight; pull `dashboard_base_url` for link substitution (tools return fully-substituted drilldown URLs).
2. `lifecycle_exposure_summary({ focus_only: true })` — scope the estate.
3. Customer chooses sunset or keep direction. *(For a modernization-prep question, work both ends inward instead of forcing one direction — see [[playbooks/application-lifecycle/lifecycle-routing]]: lead the keep-end and offer sunset as rollout-scope reduction.)*
4. **Sunset branch:** `sunset_candidates` → (AskUserQuestion) → `sunset_disposition(replicaid)` → for `sunset/archive` with >1 instance, `archival_instance_pick(replicaid)`.
5. **Keep branch:** ask the customer for `target` → `keep_candidates(target)` → (AskUserQuestion) → `keep_disposition(replicaid, target)` for the modernize cohort, or `qualify_app_analysis(replicaid, target)` for the qualify cohort.

**Interactive-follow-up rule:** when offering to drill into one app after a ranked-list tool (`sunset_candidates`, `keep_candidates`), present the choice as an `AskUserQuestion`, not as a prose question.

**Qualify-rows caveat:** the sunset funnel has a third disposition, `sunset/qualify/*` (blocked: `missing_retention_config`, `conflict_retention`, `conflict_replication`, or the combined reason). `sunset_candidates(bucket: 'qualify')` returns them; `bucket: 'delete'`/`'archive'` return only the cleanly-decided cohorts. A qualify app is **not** a delete or archive recommendation — its named issue must be resolved first.

## Pattern-ID mapping inventory (canonical checklist)

Tools return **pattern IDs**, not prose; the consultant-facing sentence is composed from the pattern. Full prose lives in the funnel pages (linked); this is the canonical completeness inventory across all five pattern families. **Append-only:** a new pattern value must be added to the wiki mapping before the MCP emits it; an unmapped value is treated as a "needs review" fallback.

**`recommended_action_pattern`** — `sunset_disposition` (prose in [[playbooks/application-lifecycle/sunset-funnel]]):
`archive_clean` · `archive_scan_failed` · `delete_clean` · `delete_scan_failed` · `configure_retention` · `reconcile_retention_config` · `resolve_replication_divergence` · `resolve_replication_and_retention`
(Bucket comes from `lifecycle_vector`; the `*_scan_failed` variants are added by the MCP scan-freshness overlay. The `missing_retention_config` / `conflict_retention` / `conflict_replication` reasons map to `configure_retention` / `reconcile_retention_config` / `resolve_replication_divergence` respectively. Retired with the column model: `delete_retention_unconfigured`, `delete_retention_incomplete`, `paused_replication_divergence`.)

**`framing_pattern`** — `archival_instance_pick` / `qualify_app_analysis.original_vs_derived` (prose in [[playbooks/application-lifecycle/sunset-funnel]]):
`oldest_surviving_instance`

**`reason_pattern`** — `keep_disposition.hand_off` (prose in [[playbooks/application-lifecycle/keep-funnel]]):
`modernize_bound_with_source_code` · `modernize_bound_no_source_code` · `modernize_none_yet` · `qualify_residual_needs_diagnosis`

**`recommended_output_pattern`** — `qualify_app_analysis` (prose in [[playbooks/application-lifecycle/keep-funnel]]):
`modernize_alongside_template` · `declining_toward_sunset` · `too_new_to_classify` · `unclassified_dormant` · `keep_no_action` · `let_it_die_when_target_lands`

**`ride_along_pattern`** — `qualify_app_analysis.template_inheritance` (prose in [[playbooks/application-lifecycle/keep-funnel]] Section 4):
`inherits_from_master_template` · `none` · `null`

## Interpretation

- `lifecycle_exposure_summary` is the aggregate per-disposition view + estate context.
- `sunset_candidates` / `keep_candidates` are the ranked working sets.
- `sunset_disposition` / `keep_disposition` are the per-app decisions, each carrying flags and a pattern ID.
- `archival_instance_pick` is the DB-instance-grain pick for `sunset/archive`.
- `qualify_app_analysis` is the heavyweight multi-layer diagnosis for the `keep/qualify` residual.
- `stakeholder_cohort` is the standalone department/location cohort for one app (same shape as the qualify cohort layer), used for `sunset/archive` context and standalone "who depends on this app?" questions.

## Fallback rule

If the MCP server is unavailable, the **base** lifecycle bucket is still readable directly from `cqx_data.replica_set.lifecycle_vector` (focus rows; see [[concepts/lifecycle-vector]]) — e.g. `SELECT lifecycle_vector, COUNT(*) FROM cqx_data.replica_set WHERE is_focus_db GROUP BY 1`. That degraded fallback gives the bucket only — not the scan-freshness overlay, the recommended-action patterns, or the per-app dispositions. Report it as a base-bucket view, lean on the iDNA dashboards as visual surfaces, and do not fabricate dispositions or a scan-failed verdict without the tools.

## Sources

Application-lifecycle cycle-2 tool routing; contract in `docs/plans/active/app-lifecycle-playbook-rewrite/app-lifecycle-playbook-rewrite.pseudo-interfaces.md`.
