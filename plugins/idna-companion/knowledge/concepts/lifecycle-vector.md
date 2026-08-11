---
id: concepts/lifecycle-vector
title: Lifecycle Vector
type: concept
status: verified
tags: [domain/idna, topic/consolidation, topic/modernization]
related:
  - concepts/remediation-classification
  - concepts/about-modernization-vector
  - concepts/usage-tier
  - schema/tables/replica-set
  - schema/tables/replica-set-summary
  - playbooks/application-lifecycle/lifecycle-routing
  - playbooks/application-lifecycle/sunset-funnel
  - playbooks/application-lifecycle/keep-funnel
  - howtos/use-application-lifecycle-mcp-tools
requires_capability: none
source: ETL dda_dm/scheduled/280_dda_dm.ai_replicaset_catalog.update.sql (lifecycle CASE + display-label CASE) + product-owner input; active variant set confirmed against live MCP
last_reviewed: 2026-07-23
---

# Lifecycle Vector

## What it is

`lifecycle_vector` on `cqx_data.replica_set` (and `cqx_data.replica_set_summary`) is the **ETL-computed application lifecycle vector** — a single column that is the source of truth shared by the iDNA dashboards and the Companion. It is computed once per ETL run, so dashboards reading `cqx_data` directly and the Companion's lifecycle MCP tools see the same value. It is **NULL for non-focus** replica sets (only focus applications are classified).

The value shape is `<direction>/<disposition>[/<reason>]`.

## Direction and dispositions

**Direction** is usage-based and matches iDNA's Consolidation Potential gate (see [[concepts/remediation-classification]]): an app goes to **sunset** only when it has **no usage in the recent 90-day window *and* no-or-very-low usage across all recorded history**. Any higher historical tier (`usage_all_time` of Low or above) *or* any recent activity (`user_access_days_last90d > 0`) sends it to **keep**. (In the ETL this is `usage_display_index = 0`: `usage_all_time` is Very Low / No Usage and `user_access_days_last90d = 0`. Note "Very Low" is non-zero — minimal historical usage still counts.)

The full active set below is grouped by direction. Each row is a value the ETL CASE can emit today, its driver, and the human-readable display label the ETL builds in `lifecycle_vector_display` (labels for `%/conflict_*` and `%/warning_check_status` are explicit; all other `/<reason>` values fall back to `<Direction>/<Disposition>: <retention_analysis_status_detail>`).

**Sunset** (`usage_display_index = 0`): the destructive path.

| Value | Driver | Display label |
|---|---|---|
| `sunset/archive` | `retention_config_status = 'ok'` AND has docs in retention window | Sunset/Archive: Consolidation Potential, but Docs in Retention |
| `sunset/archive/warning_check_outdated_contentage_scan_results` | above, plus non-empty `retention_analysis_status_lifecycle_annotation` (annotation appended) | fallback: `retention_analysis_status_detail` |
| `sunset/archive/warning_contentage_scan_failure_on_some_replicas` | above, plus non-empty annotation (annotation appended) | fallback: `retention_analysis_status_detail` |
| `sunset/delete` | consolidation potential, nothing in retention | Sunset/Delete: Consolidation Potential & No Docs in Retention |
| `sunset/delete/template_file_content_not_analyzed` | `retention_analysis_status_detail = 'Info: Not Analyzed (NTF)'` | fallback: `retention_analysis_status_detail` |
| `sunset/delete/<annotation>` | delete branch with non-empty annotation (e.g. `warning_check_outdated_contentage_scan_results`) | fallback: `retention_analysis_status_detail` |
| `sunset/qualify/missing_retention_config` | `retention_config_status = 'missing'` — configure retention before disposing | fallback: `retention_analysis_status_detail` |
| `sunset/qualify/conflict_retention` | `retention_config_status = 'conflict'` — instances disagree on retention config | Sunset/Qualify: Conflict - Diverging Retention Config between Replicas |
| `sunset/qualify/conflict_replication` | `documents_count_min != documents_count_max` — instances disagree on document count | Sunset/Qualify: Conflict - Replication Inconsistency |
| `sunset/qualify/conflict_replication+conflict_retention` | both conflicts | Sunset/Qualify: Conflict - Replication Inconsistency & Retention Config Conflict |
| `sunset/qualify/conflict_contentage_scan_failure_on_all_replicas` | `retention_analysis_status_detail LIKE 'Error:%'` (annotation `error_` rewritten to `conflict_`) | fallback: `retention_analysis_status_detail` |

**Keep** (`usage_display_index != 0`): the non-destructive path. Within keep, `usage_index_last90d >= 3` (≥ Medium recent usage) → `keep/modernize`, otherwise → `keep/qualify`.

| Value | Driver | Display label |
|---|---|---|
| `keep/modernize` | recent usage ≥ Medium (`usage_index_last90d >= 3`) | Keep/Modernize: Considerable Usage |
| `keep/modernize/conflict_replication` | modernize branch, `documents_count_min != documents_count_max` | Keep/Modernize: Conflict - Replication Inconsistency |
| `keep/modernize/warning_check_status` | modernize branch, non-empty `retention_analysis_status_lifecycle_annotation` | Keep/Modernize: Warning - Check Status |
| `keep/qualify/usage_low` | kept but recent usage < Medium, no conflict | Keep/Qualify: Low Usage |
| `keep/qualify/conflict_replication` | keep-qualify branch, `documents_count_min != documents_count_max` | Keep/Qualify: Conflict - Replication Inconsistency |
| `keep/qualify/warning_check_status` | keep-qualify branch, non-empty `retention_analysis_status_lifecycle_annotation` | Keep/Qualify: Warning - Check Status |

> **Defined in the ETL but currently disabled (commented out) — not emitted:** `keep/modernize/conflict_retention`, `keep/modernize/conflict_replication+conflict_retention`, `keep/qualify/conflict_retention`, `keep/qualify/conflict_replication+conflict_retention`. On the keep path, retention conflicts are deliberately suppressed today; only replication conflicts and content-age warnings ride along. Do not treat these four as live values.

## Blocker vs qualifier (why sunset and keep differ)

The rule is: **conflicts block destructive paths and merely qualify non-destructive ones.**

- On the **sunset** path the action is destructive (archive/delete), so any conflict or missing retention config **blocks** the disposal — the app funnels into `sunset/qualify/<reason>` and gets no archive/delete recommendation until the issue is resolved.
- On the **keep** path nothing destructive happens, so a conflict is a **qualifier** that rides on the real disposition (`keep/modernize/...`, `keep/qualify/...`).

The two meanings of `qualify` follow from this:

> `sunset/qualify/*` means "do not delete or archive yet; resolve this blocker first." `keep/qualify/*` means "kept and parked for per-app diagnosis or conflict resolution — not a destructive-action blocker."

`conflict_*` reasons take precedence over `usage_low`; once a conflict is resolved, a low-usage keep app drops to `keep/qualify/usage_low` on the next ETL run.

## How the reasons are derived

- **`retention_config_status`** = `count(DISTINCT retention_cutoff)` across the replica set's instances: `0 → missing` (no retention configured anywhere → `missing_retention_config`), `1 → ok`, `≥2 → conflict` (`conflict_retention`). Because distinct-count ignores NULLs, one configured instance among unconfigured peers is `ok`.
- **Replication divergence** (`conflict_replication`) = `documents_count_min != documents_count_max` across instances — the replicas hold different document counts.
- **Archive vs delete** (within consistent retention) = whether any instance holds documents inside the retention window.
- **Content-age / scan-driven reasons** come from the content-age signals `retention_analysis_status_lifecycle_annotation` and `retention_analysis_status_detail` — the same signals documented on [[schema/tables/replica-set]]:
  - On the **sunset** path, a `retention_analysis_status_detail LIKE 'Error:%'` (the content-age scan failed on all replicas) produces `sunset/qualify/conflict_contentage_scan_failure_on_all_replicas` — the `error_` prefix in the annotation is rewritten to `conflict_`, so it acts as a blocker. A non-empty annotation on an archive branch appends as `sunset/archive/<annotation>` (e.g. `warning_check_outdated_contentage_scan_results`, `warning_contentage_scan_failure_on_some_replicas`); on the delete branch it appends as `sunset/delete/<annotation>`. `retention_analysis_status_detail = 'Info: Not Analyzed (NTF)'` (a template that was not content-analyzed) produces `sunset/delete/template_file_content_not_analyzed`.
  - On the **keep** path, a non-empty annotation on an app being kept collapses to `keep/modernize/warning_check_status` or `keep/qualify/warning_check_status` — a qualifier, not a blocker.

## Relationship to the MCP tools

The lifecycle MCP tools read this column for the bucket and add only the consultant-workflow layer on top: recommended-action patterns, the keep hand-off matrix, stakeholder cohorts, and a **scan-freshness** overlay (from `collection_status` / `contentage_scantime`) that degrades a delete/archive recommendation to `*_scan_failed` when the content-age scan is stale. The column now carries a *coarse, cadence-fixed* scan signal of its own — the ETL appends content-age scan reasons to the disposition (e.g. `sunset/archive/warning_check_outdated_contentage_scan_results`, or the qualify blocker `sunset/qualify/conflict_contentage_scan_failure_on_all_replicas`), reflecting the content-age scan state as of the last ETL run. The MCP overlay is the **live, configurable** one (`MCP_SCAN_FRESHNESS_DAYS`, default 30): it recomputes freshness at query time from `collection_status` / `contentage_scantime`, so it can flag a scan that went stale *since* the ETL ran, and it is what drives the `*_scan_failed` recommendation. The MCP treats the ETL reason as informational only and never recomputes the disposition from it: `sunset_disposition` surfaces the archive/delete scan-reason suffix (where present) in a dedicated `etl_scan_reason` field, while qualify blockers stay visible in `bucket` and are represented by `recommended_action_pattern`. Dashboards can read those same columns directly.

## Relationship to modernization

The lifecycle vector answers *whether to keep* an app; the **modernization vector** answers *how to get the keepers green* for a compatibility target (Notes 64-bit, Nomad) — see [[concepts/about-modernization-vector]]. Modernization rows exist only for `keep/%` focus apps; sunset apps carry no modernization vector but still appear in modernization context as design-family peers and in shared-block reach. Route "is this app ready for X / what do we fix" questions to the modernization surfaces, not to this column.

## Degraded SQL fallback

If the MCP is unavailable, the **base vector** is still readable directly from `cqx_data.replica_set.lifecycle_vector` (focus rows). That gives the bucket only — not the scan-freshness overlay, the action patterns, or the per-app dispositions. Report it as a base-bucket view; never fabricate a disposition from SQL alone.

## Caveat

The vector is only as current as the last ETL run. Lifecycle decisions also depend on usage-data freshness — see the preflight in [[playbooks/application-lifecycle/lifecycle-routing]].

## Sources

ETL lifecycle CASE in `280_dda_dm.ai_replicaset_catalog.update.sql` plus product-owner input.
