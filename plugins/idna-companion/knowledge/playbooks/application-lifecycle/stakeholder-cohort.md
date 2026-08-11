---
id: playbooks/application-lifecycle/stakeholder-cohort
title: Stakeholder Cohort — Who Depends on This Application
type: playbook
status: verified
tags: [domain/idna, topic/consolidation, topic/usage, audience/consultant]
related:
  - playbooks/application-lifecycle/lifecycle-routing
  - playbooks/application-lifecycle/sunset-funnel
  - playbooks/application-lifecycle/keep-funnel
  - howtos/use-application-lifecycle-mcp-tools
  - concepts/regulatory-usage-constraints
  - concepts/about-replica-sets
  - schema/topics/department-and-location
  - schema/tables/usage-cube-repset-vs-department
  - schema/tables/usage-cube-repset-vs-location
  - howtos/analyze-usage-by-department
  - howtos/identify-content-creators-and-consumers
  - howtos/attribute-usage-aggregate-only
requires_capability: none
source:
last_reviewed: 2026-05-21
---

# Stakeholder Cohort — Who Depends on This Application

Cross-cutting overlay used by both lifecycle funnels. Identifies the people and groups that depend on a given application: which departments, which locations, and (in permissive mode only, via linked howtos) which individual users.

This page itself stays at restricted level (`requires_capability: none`). Permissive-mode details are delegated to linked howtos.

## When to retrieve

Use when the lifecycle conversation needs to answer *"who depends on this app?"*, *"who creates its content?"*, or *"who are its active users?"* for one application or a small set. Triggered from:

- [[playbooks/application-lifecycle/sunset-funnel]] for `sunset/archive` cases (knowing who created retained content is useful before committing to archive).
- [[playbooks/application-lifecycle/keep-funnel]] qualify work (Section 4) — always; and modernize disposition when stakeholder dependencies inform the modernization sequencing.

## Goal

Produce a **stakeholder cohort** for one application or a small set, with role labels where the data supports them.

A cohort, not an owner. Decade-old applications rarely have a reliable registered owner; the practical population is heavy users, originating departments and locations, and content creators vs consumers. The framing shifts from *"who owns this"* to *"who depends on this and what role do they play."*

## Operating mode

**Restricted is the default.** Department and location attribution is restricted-safe (the names are pseudonymized at iDNA's ETL layer below the user-count threshold — see [[schema/topics/department-and-location]]). Per-user attribution and content-creator-vs-consumer differentiation are permissive-only and delegated to linked howtos.

See [[concepts/regulatory-usage-constraints]] for the mode framing and the MCP-side enforcement model.

## How the cohort is surfaced

- **Standalone, and for `sunset/archive` cohort context** — call **`stakeholder_cohort(replicaid)`**. It returns `top_departments`, `top_locations`, and a `cohort_shape` for one app, callable regardless of lifecycle bucket. This is the tool for *"who depends on this app?"* / *"which departments use it?"* and for adding stakeholder context before committing a `sunset/archive` decision.
- **In the keep/qualify path** — the same cohort is already a layer of `qualify_app_analysis` (`layers.stakeholder_cohort`, identical shape); read that layer rather than making a second call when you are already running the qualify analysis.

See [[howtos/use-application-lifecycle-mcp-tools]] for the tool map. Both surfaces operate at department/location grain on `cqx_data.usage_cube_repset_vs_department` / `cqx_data.usage_cube_repset_vs_location` — both restricted-safe. See [[schema/topics/department-and-location]] for the cube structure. The **Department/Location Activity by Replica Set** dashboard (`{{idna_server}}/metabase/dashboard/10000012`) is the visual companion.

## Reading `cohort_shape`

`cohort_shape` (returned by `stakeholder_cohort`, and identically by `qualify_app_analysis.layers.stakeholder_cohort`):

| `cohort_shape` | Interpretation |
|---|---|
| `single_dominant` | One department or location holds the majority of access. Clear ownership candidate for any subsequent conversation (archive, modernize, retire). Frame as *"this app is concentrated in [department X]; that's likely your sponsor."* |
| `broadly_distributed` | Enterprise-shared. No single owner; modernization or sunset decisions need a broader steering conversation. |
| `no_data` | The usage cubes returned no department/location rows for this app — no attributable cohort (distinct from a masked name; see below). |

## Permissive drill — per-user attribution

Permissive-mode work — per-user heavy users, content-creator vs consumer split — is **not authored on this page**. The permissive content lives in dedicated howtos that themselves declare the appropriate `requires_capability`:

- [[howtos/identify-content-creators-and-consumers]] — heavy users + creator-vs-consumer differentiation.
- [[howtos/analyze-usage-by-department]] — org-attribution recipe paired with per-user output where the active session permits.

When the active access profile is permissive, these howtos surface naturally via retrieval. When the profile is restricted, the runtime filter (per [[concepts/regulatory-usage-constraints]]) prevents them from being retrieved at all.

## Output

- **Ordered cohort** — departments and locations ranked by access-day count, with active-user counts alongside.
- **Role labels** where available — content creator vs consumer counts (permissive only).
- **Masked-name note** — below iDNA's user-count threshold, a department or location appears with a **pseudonymized (masked) name** — the row and its usage counts are still present; only the name is masked. A masked-name row is **real usage that cannot be attributed to a named org unit**, not a missing row and not zero usage. Report the usage and state that the cohort cannot be named at the engagement's privacy posture.

## Interpretation

- **Single dominant department or location** — clear ownership candidate. Frame as the likely sponsor for any decision.
- **Broadly distributed** — enterprise-shared; needs a broader steering conversation.
- **Masked-name dominant cohort** — the app's heaviest cohort sits below the pseudonymization threshold. Useful signal in itself: the app is narrow enough that named attribution is inappropriate at the engagement's privacy posture. Report the usage volume; do not attempt to name the cohort.

## Sources

Workflow design from `docs/plans/active/app-lifecycle-playbook-rewrite/app-lifecycle-playbook-rewrite.md`. Tool routing per `docs/plans/active/app-lifecycle-playbook-rewrite/app-lifecycle-playbook-rewrite.pseudo-interfaces.md`.
