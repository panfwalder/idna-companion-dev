---
id: howtos/use-source-code-remediation-mcp-tools
title: Use the source-code-remediation MCP tools
type: howto
status: verified
tags: [domain/idna, topic/modernization, audience/consultant]
related:
  - playbooks/source-code-remediation/remediation-routing
  - playbooks/source-code-remediation/assess-exposure
  - playbooks/source-code-remediation/classify-template-paths
  - playbooks/source-code-remediation/classify-shared-code-paths
  - playbooks/source-code-remediation/build-remediation-plan
  - concepts/compatibility-ruleset-routing
  - concepts/notes-64bit-remediation-profile
  - concepts/nomad-web-mobile-remediation-profile
  - schema/tables/design-insight-overview-byrepset
  - schema/tables/code-source
  - playbooks/application-lifecycle/lifecycle-routing
requires_capability: none
source: mcp-server Phase 2 tool surface and remediation playbook mapping
last_reviewed: 2026-07-24
---

# Use the source-code-remediation MCP tools

## Question

"Which MCP tool should I use for this remediation question?"

## Approach

When the MCP server is available, prefer the semantic remediation tools over ad hoc SQL. The playbooks remain the "why" and "when" layer; the MCP tools are the execution layer for the most common consultant questions.

Use `execute_sql` only when:

- the active session is **permissive** (`access_profile = 'privileged'`) — `execute_sql` is privileged-only and is refused under restricted mode before any SQL is issued; in restricted mode stay with the semantic tools, or present documented SQL for the customer to run manually with their own credentials
- no existing semantic tool covers the question
- you already know the exact table and column names
- the caller needs a custom slice that is narrower or different from the semantic tool outputs

## Required inputs

- target resolved via [[concepts/compatibility-ruleset-routing]]
- either a known target shortcut such as Notes 64-bit / Nomad Web/Mobile, or an explicit `ruleset_id`
- application identifier (`replica_id`) or `code_hash_id` where relevant

## Routing map

Readiness / work-plan questions (the modernization surfaces):

- "Which apps aren't ready for `<target>`?" / "How ready are we?" / "Where do we start?" -> `modernization_plan_overview` (readiness KPI + session queue ranked by apps cleared)
- "Is app `<X>` ready / what fixes it?" -> `modernization_for_app` (replicaid or exact title)
- "What exactly do I do in session `<anchor>`?" -> `modernization_session_dossier` (session identity is the PAIR kind + anchor)
- "Which fixes do we author once and reapply?" -> `modernization_plan_overview` top blocks / the worklist ([[playbooks/source-code-remediation/analyze-shared-vs-solo-blocks]])

Exposure / finding-volume questions:

- "How big is our 64-bit or Nomad problem (findings, severity)?" -> `exposure_summary`
- "Show me the most affected apps." -> `most_affected_apps`
- "Show me the findings in app `<replica_id>`." -> `findings_for_app`
- "What should I fix first in this app?" -> `prioritize_findings`
- "Show me the normalized source for code block `<code_hash_id>`." -> `get_code_block`
- "Where else does this code block appear?" -> `find_code_block_usage` (pass `target`/`ruleset_id` for per-ruleset app reach; without it the rows are occurrence-grain across all rulesets)
- "Which template families drive the most downstream exposure?" -> `templates_by_downstream_exposure`

Readiness questions route to the modernization tools, volume/severity questions to the exposure tools — do not answer one with the other.

## Procedure

1. Bind the compatibility target first.
   Use [[concepts/compatibility-ruleset-routing]] to resolve the target profile before choosing a remediation tool.

2. Prefer the narrowest semantic tool that matches the customer question.
   Do not start with `execute_sql` if the question is already covered by a remediation tool.

3. Preserve the consulting scope in the answer.
   State whether the result is Focus applications only, all applications, one app, or one shared code block.

4. Use ad hoc SQL only as a fallback.
   If a semantic tool does not cover the needed cut, explain the gap and then use `execute_sql` with explicit intent.

## Interpretation

- `modernization_plan_overview` IS the customer-facing work plan: quote `item_display` sentences; sessions rank by `n_apps_green` (apps one sitting turns fully green), not `n_apps`; close overview answers by offering the drill-down.
- `modernization_for_app` is the per-app verdict + block work list (work states cleared_by_refresh / apply / review / scan_first; the authoring home is the pair `authored_session_kind` + `authored_in`). Sunset apps are MIS_ROUTED to `sunset_disposition` — follow it.
- `modernization_session_dossier` is a worksheet executed by the customer's developer in Domino Designer — iDNA locates and quantifies; scan_first rows are a Domino-admin ask. Diverging (review) rows carry a firm warning: a refresh overwrites them.
- All three error with NOT_FOUND + a manual-fallback hint when the ruleset is not in the modernization precompute — that means "use the manual playbook method", not "empty estate".
- `exposure_summary` is the aggregate volume view.
- `most_affected_apps` is the named application working set.
- `findings_for_app` is the bridge from application scope into concrete finding rows.
- `prioritize_findings` is the planning lens inside one application.
- `get_code_block` and `find_code_block_usage` are the shared-code drilldown pair. Ruleset-bound `find_code_block_usage` reach is raw-finding reach: an upper bound when FP suppressions are configured — the ignore-aware numbers are `modernization_worklist.reach` and the vector counts.
- `templates_by_downstream_exposure` is the template-leverage lens at the finding level.

**Lifecycle direction hint (read-only triage, not a disposition).** **When the MCP returns** `lifecycle_direction_hint` on `most_affected_apps` rows or the `findings_for_app` summary (`keep` / `sunset` / `null`), treat it as the coarse direction of the app's ETL lifecycle vector — surfaced so a remediation conversation can spot apps that may not be worth remediating. It is **not** an actionable disposition: a `sunset` hint means *consider retiring instead of fixing*. Route to the lifecycle tools (`sunset_candidates` / `sunset_disposition`, or `keep_candidates` / `keep_disposition`) for the gated decision — never act on the hint directly. `null` = non-focus or not classified. **If the field is absent** (an older MCP build that predates this field), continue the remediation flow normally and do not infer lifecycle direction. See [[playbooks/application-lifecycle/lifecycle-routing]].

## Fallback rule

If the MCP server is unavailable, fall back to the wiki's documented SQL patterns in the playbooks and schema pages. The playbooks still define the right sequence even when execution falls back to SQL.

## Sources

Phase 2 MCP implementation and remediation playbook mapping.
