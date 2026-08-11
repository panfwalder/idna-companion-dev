---
id: concepts/regulatory-usage-constraints
title: Regulatory and works-council constraints on usage analysis
type: concept
status: verified
tags: [topic/regulatory, topic/content-age, audience/consultant]
related:
  - schema/topics/department-and-location
  - playbooks/application-lifecycle/lifecycle-routing
  - playbooks/application-lifecycle/sunset-funnel
  - playbooks/application-lifecycle/keep-funnel
  - playbooks/application-lifecycle/stakeholder-cohort
  - howtos/attribute-usage-aggregate-only
  - concepts/user-access-days
  - schema/tables/usage-cube-repset-vs-user
  - schema/tables/repset-activity-summary-per-user
  - schema/tables/usage-sessions
requires_capability: none
source:
last_reviewed: 2026-07-27
---

# Regulatory and works-council constraints on usage analysis

## What it is

A framing for the **strict** vs **permissive** modes the Companion operates in across customer sites. iDNA exposes per-user activity (sessions, last-access user, per-user UAD via the session tables); whether the consultant may *use* that data depends on the customer's regulatory and labour-relations posture, not on the technical capability of the database.

Two operating modes:

- **Strict mode** — per-user and per-session analysis is prohibited. Only aggregate-grain queries (department, location, replica-set totals with minimum-group-size obfuscation already applied by iDNA) are allowed. Common at DACH/EU customers with active works councils and customers explicitly applying EU AI Act controls to "monitor employee performance" use cases.
- **Permissive mode** — the customer has signed off on per-user attribution. Per-user access ranking, per-session drill-down, and content-creator vs consumer attribution are all in scope.

Application-level usage (UAD aggregated to replica-set grain via `cqx_data.replica_set`) is identical in both modes. The split bites at the *attribution* step, not at the *ranking* step.

Tenant posture is governed by three orthogonal license flags (`pseudonymize`, `enable_mcp`, `restrict_mcp`) processed by the iDNA application; the strict/permissive distinction above is the runtime expression of `restrict_mcp`. The full matrix and its defensibility are spelled out in the "License-driven posture" subsection below.

## Why it matters for Notes/Domino / iDNA work

Application lifecycle decisions — consolidate, retire, archive, modernize — almost always require attribution to *someone*. Identifying stakeholders, departments, and content owners is load-bearing for the conversation. Running the wrong query at the wrong customer is a compliance incident, not just a faux-pas.

Three pillars sit behind the strict posture:

1. **EU AI Act constraints on automated per-individual analysis.** The Act treats "AI systems intended to be used to monitor and evaluate the performance and behaviour of persons in employment" as high-risk. Building per-user usage profiles to inform retention/retirement decisions can fall under that designation depending on how the output is used. Customers operating under AI Act compliance regimes typically restrict per-user analysis to aggregate cohorts.
2. **Works-council prohibitions on per-user productivity profiling.** In Germany, Austria, and several other EU jurisdictions, the works council (Betriebsrat / Arbeitnehmer­vertretung) has co-determination rights over technical systems that monitor employee behaviour or performance. Per-user activity reporting commonly requires explicit works-council agreement, with default posture being prohibition. Group-size minimums and aggregation are the typical compromise.
3. **Legal retention obligations independent of usage.** Tax law, sector regulators (financial services, pharma, healthcare), and contract law all impose document-retention windows independent of how often a database is opened. A dormant application with documents inside its retention window cannot be retired regardless of UAD. This is the "document-centric" branch of the application-lifecycle playbook and applies in both strict and permissive modes.

## Details

### Today: license-driven enforcement at three layers

The Companion's MCP server enforces the strict / permissive boundary at three layers under rev 6.3:

1. **Postgres role grants.** Restricted-mode tenants connect under the `cqx_role_mcp_restricted` role, which has no `SELECT` on the deny-list tables (per-user activity, MarvelClient, mail per-user, person tables). The deny list is normative in `docs/decisions/0006-appendix-restricted-allowlist.md` (dev-time reference).
2. **Manifest enforcement.** Each semantic tool declares the tables and columns it touches; the runtime parser enforces the declaration before query execution. Tools that would reference deny-listed tables cannot ship in restricted mode.
3. **`execute_sql` is privileged-only.** The ad-hoc SQL tool is registered only for permissive (`cqx_role_mcp_privileged`) sessions; under restricted mode the runtime profile gate refuses it before any SQL is issued. Restricted mode therefore still has no open-ended escape hatch — only registered, application-centric semantic tools run. (Even in permissive mode the Postgres role, not the tool, bounds table reach.)

License flags processed by the iDNA application drive role membership reassignment for the per-tenant MCP login user on a 2-minute cadence. The MCP detects changes via Postgres introspection (`pg_has_role`) and serves the new posture without restart.

**Operational caveat that survives.** The architecture controls the AI surface. A consultant or DBA running ad-hoc SQL with their own (non-MCP) credentials at a strict customer is outside the AI surface and outside what the architecture enforces. That case stays "consultant must not run X" — the protection is procedural, not technical, for the consultant-as-DBA path.

### License-driven posture: the three flags and the matrix

Three orthogonal license flags govern the MCP's data exposure for a tenant. Each does one thing.

- **`pseudonymize` (Y/N)** — existing iDNA flag. Controls whether activity-related user names (and below-threshold department/location names) are pseudonymized at the ETL layer before reaching any consumer including the MCP.
- **`enable_mcp` (Y/N)** — whether MCP is provisioned for the tenant at all. Default for new and migrating tenants is `N`.
- **`restrict_mcp` (Y/N)** — when MCP is enabled, whether the AI runs in restricted (`Y`) or permissive (`N`) mode. Default when MCP is enabled is `Y`.

The six legitimate cells:

| `pseudonymize` | `enable_mcp` | `restrict_mcp` | Tenant posture |
|---|---|---|---|
| Y | N | * | DACH default, no MCP. Existing iDNA platform behaviour. |
| Y | Y | Y | DACH default with restricted MCP. AI sees pseudonymized activity data and an explicit subset of tables. Most-defensive AI cell. |
| Y | Y | N | DACH default with permissive MCP. AI sees pseudonymized activity data with full table access. Pseudonym-frequency analysis is theoretically possible; works-council precedent accepts this. |
| N | N | * | RoW or DACH-after-agreement, no MCP. |
| N | Y | Y | Real names visible (last-modifier columns, ACL principals, agent signers) but no detailed behavioural-activity tables. Conservative AI for customers who waived pseudonymization for design reasons. **Restricted ≠ anonymous in this cell.** |
| N | Y | N | Real names + full table access. Customer has cleared all obligations. |

`restrict_mcp` and `pseudonymize` are independent controls. `restrict_mcp=Y` limits table and tool reach. `pseudonymize=Y` controls whether activity-related names are pseudonymized at the data layer. Customer-side opt-in to MCP requires acknowledgment of panagenda's AI positioning paper (commercial / legal scope; out of scope for this page).

### Restricted is not an anonymity guarantee

Restricted mode (`restrict_mcp=Y`) limits **table and tool reach**. It does **not** guarantee that no real user names appear in MCP output.

- Under `pseudonymize=Y`, activity-related user names in mixed tables are hashed at the ETL layer before reaching the MCP. Restricted mode is effectively pseudonymous for activity columns under this combination.
- Under `pseudonymize=N`, activity-related user names in mixed tables carry real values. Restricted mode still excludes detailed behavioural-activity tables (per-user cubes, sessions, MarvelClient, mail per-user, person tables), but real names of last-modifiers, ACL principals, and agent signers will appear in output that joins through allowlisted tables.

The `pseudonymize=N, restrict_mcp=Y` cell is a customer-attested combination: the customer has signed both the existing iDNA pseudonymization waiver (real names in the data layer) and the AI positioning paper acknowledgment (AI capability). Both attestations are documented; their combination is the customer's explicit choice under their own conformity assessment.

### Why this is defensible

The architecture rests on four pillars; the first three follow the established panagenda iDNA platform precedent and the fourth is strengthened by rev 6.3's enforcement layers.

1. **Single data point, not behavioural pattern.** Columns like `last_accessed_user` record one attribution per application — the most-recent reader / writer. Functionally equivalent to the "Last Modified By" field in the Domino properties dialog. Not a frequency analysis, not a time series, not a productivity metric.
2. **Application-centric context.** The user-bearing column is metadata of the application record, not an entry in a user activity table. The primary entity is the application; the user name is incidental.
3. **Operational purpose — stakeholder identification.** The consulting use case is contact identification: when an application is being considered for retirement, modernization, or remediation, the last active user is the most relevant person to consult about business continuity. This is standard IT operational practice, not employee evaluation.
4. **Query-axis constraint — structurally enforced.** User-pivoted behavioural analysis under restricted mode is blocked at every layer: the tables it needs have no `SELECT` grant for `cqx_role_mcp_restricted`, `execute_sql` is refused by the runtime profile gate before any SQL is issued, and the sole user-pivoted semantic tool (`user_activity_over_window`, privileged-only per ADR 0011) is likewise refused by an identity-keyed runtime gate before any SQL — its listing may be visible in a restricted session, but every call fails closed. The constraint is in the Postgres role and the tool API — not in a prompt instruction.

The strongest version of this argument depends on the architecture maintaining its invariants: restricted-mode semantic tools stay application-centric — no user-pivoted tools are reachable under restricted mode, and adding one requires a separate review (the review for the privileged-only `user_activity_over_window` is recorded in ADR 0011; any *restricted* exposure of a user-pivoted tool requires a new ADR and audit); `execute_sql` stays unavailable under restricted; the deny list stays current as iDNA adds new tables; the wiki sensitivity layer stays profile-aware. If any invariant weakens, the argument weakens.

### Aggregate path that stays safe under restricted mode

Department and location summary cubes, `usage_cube_repset_vs_*` aggregates, and the `users_active_*` columns on `replica_set` and `usage_summary_by_database` ship in restricted mode. iDNA's ETL layer applies threshold-pseudonymization: when `pseudonymize=Y`, departments and locations whose user count falls below a configured threshold are pseudonymized at the source. The lookup tables (`cqx_data.department`, `cqx_data.location`) where IDs resolve to names also have the threshold mechanism applied. The data layer is therefore safe by default for DACH-default tenants; no tool-layer cohort suppression is required.

For `pseudonymize=N` tenants, real department and location names appear in aggregates. This follows the same posture as the last-modifier columns above: it is the customer's chosen combination under their attestations.

The operational counterpart — which queries to use and which to avoid — lives in [[howtos/attribute-usage-aggregate-only]].

### What "stakeholders, not owners" means here

Decade-old applications rarely have an accurate registered owner. Even when one exists, the relevant population for a disposition decision is broader: heavy users, originating departments/locations, content creators vs consumers. Strict mode constrains which of those signals can be queried per-user; the framing of "stakeholders" rather than "owners" survives both modes — the conversation focus shifts from "find the owner" to "identify the stakeholder cohort."

## Related

- [[schema/topics/department-and-location]] — the aggregate-grain table family with minimum-group-size obfuscation.
- [[howtos/attribute-usage-aggregate-only]] — strict-mode operational counterpart to `howtos/analyze-usage-by-department`.
- [[playbooks/application-lifecycle/stakeholder-cohort]] — where the strict/permissive split bites in practice (per-user attribution is permissive-only and delegated to linked howtos).
- [[playbooks/application-lifecycle/sunset-funnel]] — pillar 3 (retention) applies in both modes.

## Sources

EU AI Act (Regulation (EU) 2024/1689), Article 6 and Annex III item 4(b) — high-risk classification of employment-monitoring AI. Works-council co-determination is national-law specific; in Germany see Betriebsverfassungsgesetz §87(1)6. Retention obligations are sector- and jurisdiction-specific. This page summarises the framing only — a customer's compliance team is the authoritative source for any specific deployment.
