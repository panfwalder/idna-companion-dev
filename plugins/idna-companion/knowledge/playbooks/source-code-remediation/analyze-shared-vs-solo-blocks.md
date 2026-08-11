---
id: playbooks/source-code-remediation/analyze-shared-vs-solo-blocks
title: Analyze Shared vs. Solo Code Blocks
type: playbook
status: verified
tags: [domain/idna, topic/modernization, topic/64bit, topic/nomad, audience/consultant]
related:
  - concepts/about-modernization-vector
  - concepts/about-modernization-plan
  - concepts/design-families
  - schema/tables/modernization-worklist
  - schema/tables/modernization-vector
  - howtos/query-modernization-views
  - howtos/use-source-code-remediation-mcp-tools
  - playbooks/source-code-remediation/remediation-routing
  - concepts/code-block
requires_capability: none
source: modernization-vector-integration phase-2 plan §4.10; worked reconciliation example verified live 2026-07-09
last_reviewed: 2026-07-24
---

# Analyze Shared vs. Solo Code Blocks

## When to retrieve

Use when the customer asks which fixes can be authored once and reapplied, why an app's shared/solo counts look the way they do, or how a design family's block numbers reconcile — the "fix-once leverage" analysis.

## Goal

Turn per-app shared/solo counts into a concrete authored-once/apply-elsewhere work split, with every number reconciled.

## The analysis workflow (tool-first)

1. **Enter from the app** — `modernization_for_app` (replicaid or exact title, target bound). Read `n_shared_items` / `n_solo_items` and the per-block list: each block carries its `work_state`, `scope`, kept-app `reach`, and the authoring home pair (`authored_session_kind`, `authored_in`).
2. **Classify the fix-once candidates.** Blocks whose authoring home is *elsewhere* are pastes (the fix already exists — apply it); blocks authored in *this* app's session are written here and reapplied at their other carriers. Solo blocks (`reach` 1, estate-wide solo) are genuine one-offs.
3. **Estate-wide context per block** — `find_code_block_usage` **with the target bound**: grouped one-row-per-app output whose `row_count` is the block's per-ruleset focus reach. Without the target the tool returns occurrence-grain rows spanning all rulesets — never reconcile shared/solo from that.
4. **Family drift interpretation** (unify/investigate apps): design-family membership is ≥85% *design similarity* — it tolerates code drift; hashes don't. A family member can be 8/0 shared/solo next to a 4/4 sibling because the sibling carries drifted block *versions* nobody else has. See [[concepts/design-families]].
5. **Reconcile any doubted number.** The accounting identity on every worklist row: `reach = n_cleared_by_refresh + apply + review + scan_first`. The two reach scopes: the vector's shared/solo counts use estate-wide reach across ALL focus apps **including sunset**; the worklist's `reach` counts kept apps only. A block "shared only with a sunset sibling" is shared in the strict accounting but effectively a one-off in practice — say so.

## Decision points (advisory — the customer owns these)

- **Is a cross-session block worth authoring before its sessions run?** High reach + high `n_cleared_by_refresh` says the master path clears most of it anyway; high reach with zero refresh-clearance says an early authored fix pays off at every site.
- **Does `authored_in` change the sequencing?** If app X's fixes are mostly pastes from a session already in the queue, X's effective effort drops — schedule it after its authoring session.
- **Family drift:** reconcilable drift → unify and fix once; unreconcilable → accept two masters or one-offs. Partial outcomes are legitimate exits, not failures.

## Reconciliation caveat (suppressions)

`find_code_block_usage` reach is computed from raw findings; the modernization precompute additionally applies configured false-positive suppressions. With suppressions configured, the tool's reach is an **upper bound** — the ignore-aware numbers are the worklist's `reach` and the vector counts themselves.

## Manual fallback — custom rulesets / what-if analysis

For rulesets outside the modernization precompute (or privileged ad-hoc reconciliation), the per-block reach query runs against `design_insight_overview_byrepset` with a `ruleset_id` predicate — see the three-way source choice in [[howtos/query-modernization-views]]. Do not use `code_element` for reconciliation (instance grain, no ruleset scoping); it answers "show me the code", not "how shared is this".

## Output

A per-block work split — authored-here / paste-from / review / scan-first / one-off — with reach context, ready for [[playbooks/source-code-remediation/build-remediation-plan]] sequencing.
