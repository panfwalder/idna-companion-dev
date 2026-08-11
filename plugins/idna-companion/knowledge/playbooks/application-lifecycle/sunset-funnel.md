---
id: playbooks/application-lifecycle/sunset-funnel
title: Sunset Funnel — Delete or Archive
type: playbook
status: verified
tags: [domain/idna, topic/consolidation, topic/content-age, audience/consultant]
related:
  - playbooks/application-lifecycle/lifecycle-routing
  - playbooks/application-lifecycle/keep-funnel
  - playbooks/application-lifecycle/stakeholder-cohort
  - howtos/use-application-lifecycle-mcp-tools
  - concepts/about-replica-sets
  - concepts/remediation-classification
  - concepts/lifecycle-vector
  - concepts/document-age-timestamps
  - concepts/focus-db
  - schema/tables/replica-set
  - schema/tables/database-content-retention-summary
  - schema/tables/database-content-retention
requires_capability: none
source:
last_reviewed: 2026-07-04
---

# Sunset Funnel — Delete or Archive

> **Preflight:** before walking this funnel, confirm the data-currency check in [[playbooks/application-lifecycle/lifecycle-routing]] passes. Stale usage / classification data invalidates the partition that brought apps here.

iDNA's recorded usage is treated as authoritative for this funnel.

This funnel runs on the semantic lifecycle MCP tools, which read the ETL-computed `cqx_data.replica_set.lifecycle_vector` (see [[concepts/lifecycle-vector]]) and add the workflow layer. The base bucket is also a degraded SQL fallback if the MCP is down. See [[howtos/use-application-lifecycle-mcp-tools]] for the full tool map and chaining.

## When to retrieve

Use when the consolidation conversation needs to identify retire-able applications and split them cleanly into *delete* vs *archive*. Entry comes from [[playbooks/application-lifecycle/lifecycle-routing]] when the customer's question is about sunset disposition.

## Section 1 — Anchor

The sunset gate is the `sunset/*` prefix of `lifecycle_vector` at replica-set grain — the Consolidation Potential condition: **no usage in the recent 90-day window *and* no-or-very-low usage across all recorded history** (any Low+ historical tier or any recent activity sends an app to keep instead; see [[concepts/remediation-classification]], [[concepts/lifecycle-vector]]). Apps with a `sunset/*` value are in the sunset funnel; `keep/*` apps flow to [[playbooks/application-lifecycle/keep-funnel]]; non-focus apps are NULL (not classified).

- **The cohort** — call `sunset_candidates` (scope defaults to Focus replica sets). It returns the ranked sunset cohort, each row carrying its `bucket` (`sunset/archive`, `sunset/delete`, or a `sunset/qualify/*` reason), usage signal, replicas count, and per-app flags. When offering a drill-down after the ranked return, follow the interactive-follow-up rule: present the choice as an `AskUserQuestion`.
- **One app** — call `sunset_disposition(replicaid)` for the full per-app decision.

> **Qualify rows block disposal:** a `sunset/qualify/*` app is a sunset candidate whose delete-vs-archive decision is **blocked** — no retention config (`missing_retention_config`), conflicting retention config (`conflict_retention`), or divergent replicas (`conflict_replication`). The tool does not recommend delete or archive for these; the consultant resolves the named issue first.

iDNA's curated **Consolidation Potential** dashboard (`{{idna_server}}/metabase/dashboard/10000017`) is the consultant's visual companion — it shows the same candidate cohort. `lifecycle_vector` is the partition rule; the dashboard is for visual confirmation.

## Section 2 — Delete-vs-archive split

The retention decision works at **replica-set grain**, aggregating across instances. No single reference DB is selected; the question is *"does this replica set have a retention obligation?"*, not *"what does instance X say about retention?"* `sunset_disposition` performs the aggregation and returns the disposition plus the flags below. The consultant's job is to interpret them — especially the cases where the tool deliberately does **not** auto-decide.

### Retention determination (from the column)

The delete-vs-archive split is decided in the ETL via `retention_config_status` (`count(DISTINCT retention_cutoff)` across instances; see [[concepts/lifecycle-vector]]) and surfaced in `lifecycle_vector`. Two qualify reasons block the split:

- **`sunset/qualify/missing_retention_config`** (no instance has retention configured) → *"Retention values are sourced from iDNA Custom DB Properties. Without a configured retention horizon, every sunset candidate looks delete-able, but legal/compliance documents may be in scope."* Configure retention, then re-run; do **not** treat as delete.
- **`sunset/qualify/conflict_retention`** (instances disagree — ≥2 distinct retention cutoffs) → reconcile the divergent configs before deciding; the disagreement is itself a finding.

(There is no longer a separate "retention incomplete" state — one configured instance among unconfigured peers counts as configured.)

### Split logic (consistent retention)

When retention is configured consistently (`retention_config_status = 'ok'`):

- **`sunset/archive`** — some instance holds documents inside the retention period (`retention_cutoff IS NOT NULL AND docs_count_inret > 0`). `retention_cutoff` is a backward window boundary (documents modified after it are in retention), so `docs_count_inret > 0` is the lock signal — not a future-cutoff test.
- **`sunset/delete`** — retention configured, no in-retention documents.

`sunset_disposition` reads the column bucket and adds the scan-freshness overlay (below). The column-level arithmetic behind the horizon lives on [[schema/tables/database-content-retention-summary]] as reference.

### Replication divergence

When instances of the same replica set disagree on document count (`documents_count_min != documents_count_max`), the app carries **`sunset/qualify/conflict_replication`** (or `conflict_replication+conflict_retention` if retention also conflicts). **The disposal is blocked** — divergence is itself a finding, and the lifecycle disposition must not be committed until the disagreement is explained. The tool does not pick a side; the consultant investigates.

### Archival-instance pick

When the replica set lands in `sunset/archive` **and** has more than one instance, call `archival_instance_pick(replicaid)` to choose the archival target. A single-instance archive needs no pick — the lone instance is the target.

- The tool returns the **oldest surviving instance** (earliest creation-in-this-file timestamp) as `chosen`, with a per-instance drilldown link.
- It is the best archival target, **not provably the historical original** — older instances may have been decommissioned. Frame the result to the customer per the `framing_pattern` (see mapping below), never as *"the original."*
- **`flags.scan_freshness_failed`** on the pick means the chosen instance's content-age scan is stale or failed; the creation-timestamp basis for the pick may not reflect reality. Carry a confidence caveat and request a fresh scan before committing the archival target.
- Calling `archival_instance_pick` on a non-`sunset/archive` app returns a mis-route error, with `suggested_tool` pointing at the right tool — it will not fabricate an archival target for an app that should not be archived.

See [[concepts/document-age-timestamps]] for the timestamp semantics.

### Content-age data quality check (critical for sunset/delete)

Content-age data is scan-derived (typically every second Saturday). The same scan-freshness gate applies to retention reads, not only content-age reads. `sunset_disposition` sets **`flags.scan_freshness_failed`** when the relevant instance's collection failed (Error/Warning) or the scan is stale.

- **Critical for `sunset/delete`:** a stale/failed scan means the content-age signal may no longer reflect reality. Flag the candidate and request a fresh scan rather than acting on outdated data. The operational consequence of acting on stale data here is *"delete an app that shouldn't be deleted."*
- **Less critical for `sunset/archive`:** those documents are retained regardless; stale data leads to *"archive,"* not *"delete incorrectly."*

The **Retention Catalog** dashboard (`{{idna_server}}/metabase/dashboard/10000040`) is the visual companion for scan status (`Collection Status`, `Scan Time`).

### Stakeholder cohort for sunset/archive

For `sunset/archive` cases, surface the stakeholder cohort before committing — call `stakeholder_cohort(replicaid)` (see [[playbooks/application-lifecycle/stakeholder-cohort]]). Knowing which departments created or owned the retained content is useful context for the archival decision and any subsequent legal-hold conversation.

## Recommended-action patterns

`sunset_disposition` returns a `recommended_action_pattern` (a pattern ID, not prose). Compose the consultant-facing sentence from the pattern:

| `recommended_action_pattern` | Consultant-facing meaning |
|---|---|
| `archive_clean` | Archive — retention horizon in force with documents inside it. Call `archival_instance_pick` to choose the target (when >1 instance). |
| `archive_scan_failed` | Archive, but content-age data quality is flagged — request a fresh content-age scan before committing the archival instance. |
| `delete_clean` | Delete — retention configured, no in-retention documents. Confirm with the customer before committing. |
| `delete_scan_failed` | Delete candidate, but the content-age scan is stale/failed — request a fresh scan before committing. |
| `configure_retention` | Blocked (`sunset/qualify/missing_retention_config`) — retention is **not configured** for any instance. Configure it in iDNA Custom DB Properties, then re-run; do not assume delete-safe. |
| `reconcile_retention_config` | Blocked (`sunset/qualify/conflict_retention`) — instances disagree on retention config. Reconcile before deciding. |
| `resolve_replication_divergence` | Blocked (`sunset/qualify/conflict_replication`) — replicas hold different document counts. Resolve the divergence (itself a finding) before committing. |
| `resolve_replication_and_retention` | Blocked (`sunset/qualify/conflict_replication+conflict_retention`) — both a replication and a retention-config conflict. Resolve both. |

`archival_instance_pick` returns a `framing_pattern`:

| `framing_pattern` | Consultant-facing meaning |
|---|---|
| `oldest_surviving_instance` | *"Oldest surviving instance, best archival target — not provably the historical original."* Older instances may have been decommissioned. |

## Section 3 — Output

- **sunset/delete** — replica set ID, `scan_freshness_failed` confidence flag, and the path-of-least-resistance recommendation when clean.
- **sunset/archive** — replica set ID, chosen archival instance (when >1 instance) with its drilldown link, retention horizon, stakeholder cohort summary.
- **Side outputs (the `sunset/qualify/*` blockers):**
  - Replication-divergence findings (`conflict_replication`).
  - Retention-configuration gaps (`missing_retention_config`) and conflicts (`conflict_retention`).
  - Scan-freshness failures (delete candidates lacking confident data).

## Reference dashboards

Visual companions to the tools (not the decision mechanism):

- **Consolidation Potential** (`{{idna_server}}/metabase/dashboard/10000017`) — sunset candidate cohort.
- **Retention Catalog** (`{{idna_server}}/metabase/dashboard/10000040`) — retention properties, `Collection Status`, `Scan Time` per DB.

## Sources

Workflow design from `docs/plans/active/app-lifecycle-playbook-rewrite/app-lifecycle-playbook-rewrite.md`. Tool routing per `docs/plans/active/app-lifecycle-playbook-rewrite/app-lifecycle-playbook-rewrite.pseudo-interfaces.md`.
