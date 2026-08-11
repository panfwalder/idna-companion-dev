---
id: playbooks/application-lifecycle/keep-funnel
title: Keep Funnel — Modernize or Qualify
type: playbook
status: verified
tags: [domain/idna, topic/consolidation, topic/modernization, topic/usage, audience/consultant]
related:
  - playbooks/application-lifecycle/lifecycle-routing
  - playbooks/application-lifecycle/sunset-funnel
  - playbooks/application-lifecycle/stakeholder-cohort
  - playbooks/source-code-remediation/remediation-routing
  - howtos/use-application-lifecycle-mcp-tools
  - concepts/about-replica-sets
  - concepts/usage-tier
  - concepts/usage-time-windows
  - concepts/remediation-classification
  - concepts/lifecycle-vector
  - concepts/compatibility-ruleset-routing
  - concepts/about-template-inheritance
  - concepts/document-age-timestamps
  - concepts/focus-db
  - schema/tables/replica-set
requires_capability: none
source:
last_reviewed: 2026-06-19
---

# Keep Funnel — Modernize or Qualify

> **Preflight:** before walking this funnel, confirm the data-currency check in [[playbooks/application-lifecycle/lifecycle-routing]] passes. Stale usage / classification data invalidates the partition that brought apps here.

Apps in this funnel carry a `keep/*` `lifecycle_vector` — they are not sunset candidates. The funnel splits them into `keep/modernize` (worth investing in) and `keep/qualify/usage_low` (residual; per-app diagnosis required); a config/replication conflict surfaces as a `/conflict_*` qualifier on either side (resolve it before relying on the disposition). See [[concepts/lifecycle-vector]].

This funnel runs on the semantic lifecycle MCP tools, which read the ETL-computed `lifecycle_vector` and add the workflow layer. The base bucket is also a degraded SQL fallback if the MCP is down. See [[howtos/use-application-lifecycle-mcp-tools]] for the full tool map and chaining.

## When to retrieve

Use when the consolidation conversation needs to handle the active estate — *"what about everything else?"* after the sunset funnel has done its work, or when the customer's question is about modernization rather than sunset (*"which apps need preparing for 64-bit?"*, *"how do we get our active apps Nomad-ready?"*).

## Section 1 — Target state

Ask the customer for the forward **intent**, then map it to the `target` the keep tools take:

| Customer intent | `target` value |
|---|---|
| HCL Notes 64-bit | `notes-64bit` |
| Nomad Web *or* Nomad Mobile | `nomad-web-mobile` (one value — Web and Mobile share one ruleset; see [[concepts/compatibility-ruleset-routing]]) |
| Both 64-bit and Nomad | `multiple` |
| Not decided yet | `none-yet` — the dominant case in early-consolidation engagements |

`none-yet` is a **switch, not a gate**: apps still get bucketed; what differs is the downstream disposition (Section 3) and the qualify depth (Section 4).

**Only intent is asked of the customer.** Environment facts — source-code-access license, dashboard base URL — are auto-detected from `get_idna_session` and surfaced on the keep tools as `source_code_access_available` / `dashboard_base_url`. The playbook does not ask the consultant about these.

**This section owns target-binding for the lifecycle path.** The persona's "bind the target first" rule is scoped to *remediation* questions; a modernization-prep question that entered via [[playbooks/application-lifecycle/lifecycle-routing]] (e.g. *"which DBs are most important to prep for 64-bit?"*) binds its `target` here, from customer intent — `notes-64bit` when the question already names the target.

## Section 2 — Bucket the active estate

Call `keep_candidates(target, scope=focus_only)`. It returns the keep cohort ranked, each row carrying its `bucket` (from `lifecycle_vector`), usage signals across horizons, template inheritance, and (when applicable) per-ruleset findings. The partition (computed in the ETL, identical in restricted and permissive mode):

- `usage_last90d >= 'Medium'` → **`keep/modernize`** (posture: worth investing in, regardless of whether findings exist today)
- everything else → **`keep/qualify/usage_low`** (residual)
- a config/replication conflict overrides the reason on either side → **`keep/.../conflict_retention`**, **`/conflict_replication`**, or **`/conflict_replication+conflict_retention`** — resolve the conflict before relying on the disposition.

Label = posture, not current action. Non-focus apps carry NULL `lifecycle_vector` and are **not** in the keep funnel at all (out of lifecycle scope) — there is no separate "unclassified" bucket.

The qualify residual is intentionally heterogeneous — low-but-nonzero usage, No-Usage-90d apps that weren't classified Consolidation Potential, and apps with too short a history to classify. Section 4 diagnoses each per-app rather than splitting the bucket at routing time. `usage_last90d` is the fast-routing default; apps where 90d disagrees with 365d or all-time are diagnosed in qualify (`keep_candidates` sets `flags.declining_signal` / `flags.no_usage_90d` to mark them), **not** auto-promoted into modernize. See [[concepts/usage-tier]]. When the customer defines their own observation period instead ("18 months", "since the last reorg", a fiscal year), custom windows are available at weekly grain — [[concepts/usage-time-windows]] — as *diagnostic enrichment*: the funnel's routing thresholds stay defined on the canonical horizons.

When offering which app to drill into after a ranked `keep_candidates` return, follow the interactive-follow-up rule (`AskUserQuestion`). For a **modernization-prep** question, the keep cohort came from the lifecycle vector — surface that lens in the answer (see persona) and, when the displayed app list differentiates by bucket, show the bucket as a column. The natural complement is the **sunset** side: retiring unused apps shrinks the rollout surface before any fix (see [[playbooks/application-lifecycle/sunset-funnel]]). Mention or offer it where it would shape the recommendation; don't dump it.

### Working surfaces for the active estate

There is no curated "Modernization Candidates" dashboard equivalent to Consolidation Potential. Visual companions to the tools:

- **Database Catalog** (`{{idna_server}}/#/catalog`) and **Replica Set Catalog** (`{{idna_server}}/#/masterlistreplicasets`) — highly filterable across usage tiers, design information, content age, and findings.
- **Department/Location Activity by Replica Set** (`{{idna_server}}/metabase/dashboard/10000012`) — per-app stakeholder surface; used by qualify (Section 4) and historic-pattern investigation.

## Section 3 — `keep/modernize` disposition

Call `keep_disposition(replicaid, target)` for one app. Its `hand_off` field encodes the matrix below, driven by the customer-stated target and the MCP-reported `source_code_access_available`:

| Target | `source_code_access_available` | `hand_off.action` / `reason_pattern` | Disposition |
|---|---|---|---|
| Bound (`notes-64bit` / `nomad-web-mobile` / `multiple`) | `true` | `remediation_routing` / `modernize_bound_with_source_code` | Hand off to [[playbooks/source-code-remediation/remediation-routing]] with the bound ruleset(s) |
| Bound | `false` | `record_keeper_no_remediation` / `modernize_bound_no_source_code` | Record as keeper; flag the license limit (see wording note) |
| `none-yet` | any | `record_keeper_none_yet` / `modernize_none_yet` | Record as keeper; defer to a future phase |

**`reason_pattern` → consultant-facing meaning:**

| `reason_pattern` | Meaning |
|---|---|
| `modernize_bound_with_source_code` | Target bound and source-code analysis available — hand off to remediation routing with the ruleset(s). |
| `modernize_bound_no_source_code` | Target bound but **iDNA's license does not include source-code analysis** — modernization assessment is limited to usage signals; no findings to estimate effort against. |
| `modernize_none_yet` | No target chosen yet — record as keeper, revisit when the customer commits to a target. |
| `qualify_residual_needs_diagnosis` | App is in `keep/qualify`; call `qualify_app_analysis` for the per-app diagnosis. |

**Three-state source-code wording (important):** reserve the message *"iDNA license doesn't include source-code analysis"* for a **positively detected** `source_code_access_available: false`. When the signal is merely absent, the tool defaults it to `true` (optimistic); do not assert a license limitation on an absent signal — if in doubt, say the source-code state is unconfirmed and recommend verifying iDNA's configuration.

The *"no source-code access"* and *"none-yet"* cases produce operationally identical output (keepers without active remediation work) for different reasons.

**Estate-wide implication when `source_code_access_available = false`:** the entire downstream remediation conversation is degraded, not just the hand-off. The qualify per-app analysis in Section 4 returns `analysis_depth: 'light'` even when the customer has a target bound — there is no findings data to estimate effort against.

**Multi-target (`target = 'multiple'`):** `keep_disposition` returns `ruleset_findings[]` with one entry per ruleset (Notes 64-bit + Nomad), each with its own counts. Read the per-ruleset split rather than a single blended number so the customer sees where the effort actually lands.

### Reference dashboard for the hand-off

**Insights Findings by Code Element in Database** (`{{idna_server}}/metabase/dashboard/10000036`) — visual companion, filterable by Database, Server, Ruleset, Insight, and Code Hash ID. The remediation playbook owns the deeper analysis.

## Section 4 — `keep/qualify` per-app work

Call `qualify_app_analysis(replicaid, target)`. It returns the layered analysis and a `recommended_output_pattern`. **Analysis depth** is set by the tool:

- **`analysis_depth: 'full'`** (target bound AND `source_code_access_available: true`): usage trend across horizons, template inheritance (ride-along), per-ruleset findings, content age, original-vs-derived replica, stakeholder cohort. Output: near-binary recommendation against effort.
- **`analysis_depth: 'light'`** (`none-yet`, or `source_code_access_available: false`): drops the findings layer only — `template_inheritance`, `content_age`, `original_vs_derived`, and `stakeholder_cohort` still populate. No effort estimate.

### No-Usage-90d diagnostic

For apps with zero/near-zero 90d usage that weren't classified Consolidation Potential, `qualify_app_analysis` sets `usage_trend.diagnostic` to one of:

- **`declining_toward_sunset`** — `>= Medium` historical (365d/all-time), recent zero. The most common case.
- **`too_new_to_classify`** — short recorded history (derived from `db_usage_history_start_date` / absent all-time usage); insufficient data for iDNA's sunset gate.
- **`unclassified_dormant`** — neither of the above. Flag for the consultant.

(A defensive `active` value can appear if the app is actually `keep/modernize`; the tool returns a mis-route error (with `suggested_tool`) in that case rather than a diagnosis — re-route to `keep_disposition`.)

### `recommended_output_pattern` → consultant-facing meaning

| `recommended_output_pattern` | Meaning |
|---|---|
| `modernize_alongside_template` | Inherits from a master template that is itself being modernized — ride along at near-zero net effort. See [[concepts/about-template-inheritance]]. |
| `declining_toward_sunset` | Historically heavy, recently zero — re-evaluate next consolidation pass. If you suspect periodic / seasonal use rather than decline, verify via the Department/Location Activity by Replica Set dashboard (`{{idna_server}}/metabase/dashboard/10000012`). |
| `too_new_to_classify` | Short recorded history — keep, re-evaluate next consolidation pass. |
| `unclassified_dormant` | Neither declining nor too-new — needs consultant review. |
| `keep_no_action` | Flat usage, no findings cost to weigh, no migration risk — keep, no action. |
| `let_it_die_when_target_lands` | No ride-along, findings cost exists, no stakeholder dependency — let it retire when the target migration lands (target-bound full-depth only). |

### `ride_along_pattern` (template-inheritance layer)

`qualify_app_analysis.layers.template_inheritance.ride_along_pattern`:

| `ride_along_pattern` | Meaning |
|---|---|
| `inherits_from_master_template` | The app inherits from a template acting as a master template — a ride-along modernization candidate (the strong input to `modernize_alongside_template`). |
| `none` | Has template inheritance, but no master-template ride-along path. |
| `null` | No template inheritance recorded. |

## Section 5 — Output

Target-state annotated per app. For each app in the keep funnel:

- bucket label (`keep/modernize` or `keep/qualify/usage_low`, with any `/conflict_*` qualifier)
- disposition / output pattern (from the `reason_pattern` or `recommended_output_pattern` above)
- effort estimate when target-bound with source-code access; otherwise *"deferred — no ruleset / no findings data"*
- per-ruleset findings split when `target = 'multiple'`
- stakeholder cohort when relevant

## Sources

Workflow design from `docs/plans/active/app-lifecycle-playbook-rewrite/app-lifecycle-playbook-rewrite.md`. Tool routing per `docs/plans/active/app-lifecycle-playbook-rewrite/app-lifecycle-playbook-rewrite.pseudo-interfaces.md`.
