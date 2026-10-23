---
id: howtos/find-code-patterns
title: Find a code pattern in the estate - existing insights first
type: howto
status: verified
tags: [domain/idna, topic/modernization]
related:
  - concepts/shipped-insight-catalog
  - concepts/insights-and-rulesets
  - howtos/find-design-dashboards
  - howtos/use-source-code-remediation-mcp-tools
  - howtos/navigate-code
  - schema/tables/design-insight
  - schema/tables/design-insight-ruleset
  - schema/tables/design-insight-ruleset-member
  - schema/tables/design-insight-overview-byrepset
  - schema/topics/insights-overview-exclude
  - schema/topics/trial-license-masking
requires_capability: none
source: ETL repo dda_ci/080-085 (shipped ruleset ids, 3df3647); MCP finding tools (ruleset_id override verified on the panagenda test instance, 2026-10-18); cqx_data views per ETL ADR 0013; docs/plans/active/insight-catalog/; search_code routing per docs/plans/active/code-navigation (2026-10-22)
kbase_sources: [portal-insights, portal-custom-insights-manage, portal-databases]
last_reviewed: 2026-10-22
---

# Find a code pattern in the estate - existing insights first

## Question

"Do we have X in our code?" / "How can I find out whether our applications use X?" - for example hard-coded IP addresses, ODBC connections, OLE automation or mail sending.

## Approach

First check whether iDNA already detects X with a shipped or custom insight: its findings exist for every analysed application already. Only when no insight fits, search the code (Custom Insights › Code Search) and, if the pattern is worth keeping, save it as a custom insight.

## Data needed

[[concepts/shipped-insight-catalog]] (shipped insights and their rulesets); `cqx_data.design_insight`, `cqx_data.design_insight_ruleset`, `cqx_data.design_insight_ruleset_member` (live catalog, custom items); `cqx_data.design_insight_overview_byrepset` (findings).

## Procedure

1. **Is there an insight for X?**
   - Look X up in [[concepts/shipped-insight-catalog]] by name and by "What it flags".
   - The customer's own insights depend on the session:
     - privileged session: query the live catalog (below); `custom = true` marks the customer's items.
     - restricted session: this session cannot list them. Point to Custom Insights › Manage (`{{idna_server}}/#/custominsights`; the Hide toggle shows custom items only) and offer the catalog query for the customer to run.
2. **If an insight exists, show where its results are.**
   - Design › Insights (`{{idna_server}}/#/insights`): open the ruleset, then the insight, and drill into databases and code elements.
   - Insights Findings by Code Element in Database (`{{idna_server}}/#/dashboard/10000036`) or by Unique Code Block (`{{idna_server}}/#/dashboard/10000038`) with the Insight filter set; the full list can be exported.
   - Catalog › Databases (`{{idna_server}}/#/catalog`) with the column set of the insight's ruleset.
   - MCP, any profile: `exposure_summary` / `most_affected_apps` with the ruleset's `ruleset_id` (table below) rank applications by ALL insights of that ruleset, not only this one; `findings_for_app` (replica id + `ruleset_id`) lists one application's findings, each with `finding_name` = the insight name.
   - MCP, privileged: per-insight counts with `execute_sql` (below).
3. **If no insight fits:** MCP, any profile: `search_code` (mode `regex` for an expression; MCP builds with the tool) - see [[howtos/navigate-code]]. In the iDNA UI, or on a build without the tool: test a regular expression in Custom Insights › Code Search (`{{idna_server}}/#/codesearch`). A useful expression can be saved there as a pattern and assigned to a custom insight in a custom ruleset; its findings appear after the next nightly run.

### Shipped ruleset ids (identical on every iDNA server that ships the ruleset)

| Ruleset | `ruleset_id` |
|---|---|
| Cloud Readiness | `0b6d8d4e-112f-1740-dfbf-4920af0659db` |
| Dependencies on other Databases | `5a71d549-f099-6ede-5654-692cd512d3b6` |
| Mail Dependencies | `ed1fcd35-3785-56f5-0185-ecb808b2d5c2` |
| Nomad Web, iOS, Android Compatibility | `a8efac35-3e54-48f4-aa27-4c342529ed0a` |
| Notes 64bit Readiness | `71f16a9a-8fc6-4dfe-add8-3a43c539d353` |
| Operating System Dependencies | `d180b6e6-f9d6-a61c-54f2-99ad41f68a41` |
| Special Interfaces | `36c74b44-dae9-5f93-a4dd-bedc213029ab` |
| Specific Code Findings | `ba6f0d25-e8ca-2045-ffce-157e7b1af4d3` |
| Web Design | `c0df13a1-4b30-c1fe-c29f-c5124019feb1` |

### Live catalog lookup (privileged, or for the customer to run)

Insights whose name contains a term, with their rulesets (on iDNA builds with `cqx_data.design_insight_ruleset_member`; check with `to_regclass('cqx_data.design_insight_ruleset_member')`):

```sql
SELECT
    i.insight_id,
    i.name,
    i.severity,
    i.custom,
    r.ruleset_id,
    r.name AS ruleset_name,
    r.enabled
FROM
    cqx_data.design_insight i
LEFT JOIN
    cqx_data.design_insight_ruleset_member m ON m.insight_id = i.insight_id
LEFT JOIN
    cqx_data.design_insight_ruleset r ON r.ruleset_id = m.ruleset_id
WHERE
    i.name ILIKE '%<term>%'
ORDER BY
    r.name,
    i.name;
```

A NULL ruleset means the insight is in no ruleset; `enabled = false` means its ruleset is not evaluated. On builds without the membership view, the pairs show only for insights with findings:

```sql
SELECT DISTINCT
    ruleset_id,
    ruleset_name,
    insight_id,
    finding_name
FROM
    cqx_data.design_insight_overview_byrepset
WHERE
    finding_name ILIKE '%<term>%';
```

### Per-insight counts (privileged)

Applications, code blocks and occurrences of one insight in one ruleset:

```sql
SELECT
    count(DISTINCT replicaid) AS applications,
    count(DISTINCT code_hash_id) AS code_blocks,
    count(*) AS occurrences
FROM
    cqx_data.design_insight_overview_byrepset
WHERE
    insight_id = '<insight_id>'
    AND ruleset_id = '<ruleset_id>';
```

Per application: the same filter, `GROUP BY replicaid, title`, ordered by occurrences.

## Interpretation

- **Name the unit.** Occurrences count every location; code blocks count distinct code; applications count replica sets. The insights summary views count distinct (insight, code block) pairs per application - never mix the units.
- **One detection, one count.** An insight with "Same detection as" in the catalog reports the same code as its twin in another ruleset: filter on one `ruleset_id`, never add the rulesets.
- **Scope.** `design_insight_overview_byrepset` holds focus replica sets only; say so.
- **Trial licence.** What the finding rows show depends on the iDNA build: newer builds mask the rows of applications outside the trial selection (aliased names, no Designer links; counts complete, detail withheld), older builds return no finding rows - see [[schema/topics/trial-license-masking]]. Identify applications by `replicaid`.
- **Zero is not proof.** A pattern finds only the shapes it was written for - Hardcoded IP Addresses, for example, finds an address only directly after a quote, apostrophe, slash or backslash, not one inside longer text; the catalog's "What it flags" says what each insight covers. For a full sweep, test a broader expression with `search_code` (MCP builds with the tool) or in Code Search. An empty finding result is not missing code: to see whether code exists or who uses it, follow [[howtos/navigate-code]]. Databases flagged "Insights Overview Exclude" carry no findings ([[schema/topics/insights-overview-exclude]]); an insight in no enabled ruleset is not evaluated; values kept outside the code (documents, notes.ini) are not scanned.
- **Patterns are heuristics.** A finding says the pattern matched; review the code before calling it a defect.

## Variations

- "Which applications send mail / use ODBC / automate Office?" - the same steps with the matching insight (Mail Dependencies, Special Interfaces, Operating System Dependencies).
- "Does iDNA check for X at all?" - step 1 alone.
- Remediation of Notes 64-bit or Nomad findings follows [[concepts/compatibility-ruleset-routing]] instead.
