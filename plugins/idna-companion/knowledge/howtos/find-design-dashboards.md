---
id: howtos/find-design-dashboards
title: Find the Design and Custom Insights dashboards and pages
type: howto
status: verified
tags: [domain/idna, topic/dashboards, audience/customer, audience/consultant, topic/64bit, topic/nomad]
related:
  - concepts/database-complexity
  - concepts/design-similarity
  - concepts/code-block
  - playbooks/source-code-remediation/assess-exposure
  - playbooks/source-code-remediation/remediation-routing
  - concepts/shipped-insight-catalog
  - howtos/find-code-patterns
requires_capability: none
source: Portal and Metabase inventory, iDNA 3.5.0, 2026-09-08
kbase_sources:
  - metabase-10000019-agent-list-search
  - metabase-10000035-code-search-results-overview
  - metabase-10000036-insights-findings-by-code-element-in-database
  - metabase-10000037-code-block-overview
  - metabase-10000038-insights-findings-by-unique-code-block
  - portal-agent-list-search
  - portal-code-search
  - portal-complexity
  - portal-custom-insights-manage
  - portal-insights
  - portal-insights-findings-by-code-element-in-database
  - portal-insights-findings-by-unique-code-block
  - portal-similarity
  - portal-source-code
last_reviewed: 2026-10-18
---

# Find the Design and Custom Insights dashboards and pages

This page covers two Portal sections. The Design section shows the application estate from the design side: complexity, source code figures, design similarity, Insights findings and agents. The Custom Insights section is where collected code is searched and the Insights rulesets are maintained. Two related dashboards that open only in Metabase are listed here as well. Names and navigation paths are as seen in iDNA 3.5.0; older versions may differ. In links, replace `{{idna_server}}` with `dashboard_base_url` from `get_idna_session` when it is present; otherwise show the path with the placeholder and say that it stands for the customer's iDNA host.

### Complexity

- **Where:** Design › Complexity, `{{idna_server}}/#/complexity`.
- **What you get:** one bar per analyzed focus database, its height set by the design complexity index and its color by complexity class, from Very Low to Very High. Reference lines mark HCL standard templates as a yardstick, and the templates shown are selectable (Selection of Standard Templates).
- **Fits questions like:** how complex are our applications compared with the HCL standard templates? Which focus databases have the most complex design? How is design complexity spread across the estate?

### Source Code

- **Where:** Design › Source Code, `{{idna_server}}/#/complexity-code`.
- **What you get:** key figures for the source code collected across the estate (designs found, design documents, lines of code, code blocks and the Code Duplication percentage), the split by coding language, and for each design element type its element count next to its lines of code. The page has no filters.
- **Fits questions like:** how much code is in our focus databases? Which coding languages do our applications use? Is a lot of our code repeated across applications?

### Similarity

- **Where:** Design › Similarity, `{{idna_server}}/#/similarity`.
- **What you get:** databases with closely matching designs, grouped into clusters drawn as bubbles sized by member count, with key figures for cluster sizes (In Clusters >1, >10 and >100 Members). You can drill from a cluster to its sub-clusters and on to the member databases.
- **Fits questions like:** which of our databases have nearly identical designs? Are there groups of applications that could share one design? Do many of our databases belong to large design clusters?

### Insights

- **Where:** Design › Insights, `{{idna_server}}/#/insights`.
- **What you get:** a treemap of the Insights findings across the focus databases, grouped by ruleset (the shipped rulesets are listed in [[concepts/shipped-insight-catalog]]; custom ones appear alongside) and finding; tile area reflects how many findings there are, color reflects severity. Four key figures (analyzed designs, databases with findings, total findings, highest insight score) frame the chart, and from a finding you can drill down to the databases and code elements behind it.
- **Fits questions like:** which 64-bit or Nomad compatibility findings does our estate have? Which rulesets produce the most findings? How many databases have findings at all? Does iDNA already flag a pattern such as hardcoded IP addresses, and where? (Recipe: [[howtos/find-code-patterns]].)

### Insights Findings by Code Element in Database

- **Where:** Design › Insights Findings by Code Element in Database, `{{idna_server}}/#/dashboard/10000036`.
- **What you get:** a table of the Insights findings in the focus databases, listed by code element and database; you can filter by Database, Server, Ruleset, Insight and Code Hash ID. Without a filter the list can be slow to load, and the complete list can be exported from the Metabase view (Explore).
- **Fits questions like:** which code elements in this database have findings? Where in the focus databases does a particular insight occur, for example Hardcoded IP Addresses (insight names: [[concepts/shipped-insight-catalog]])? Can I get the full list of findings as an export?

### Insights Findings by Unique Code Block

- **Where:** Design › Insights Findings by Unique Code Block, `{{idna_server}}/#/dashboard/10000038`.
- **What you get:** the Insights findings organized by unique code block (Code Hash ID) instead of by code element in each database. The filters are the same: Database, Server, Ruleset, Insight and Code Hash ID.
- **Fits questions like:** which code blocks are behind most of our findings? Which findings does one code block carry? Which unique code blocks trigger a particular insight?

### Code Block Overview

- **Where:** `{{idna_server}}/metabase/dashboard/10000037`. Not in the Portal menu; it opens in Metabase and depends on your Metabase access.
- **What you get:** one code block, chosen by its Code Hash ID (the dashboard stays empty without one): how many replica sets and database instances it affects, its number of findings, its effective lines of code and any suspected standard template relation, plus lists of its findings, the affected code elements and the templates related to the affected databases. Links on the dashboard lead on to the block's findings in focus databases and in master templates and to a preview of its source code.
- **Fits questions like:** how many applications contain this code block? Which templates are related to the databases that carry it? What does this code block look like?

### Agent List Search

- **Where:** Design › Agent List Search, `{{idna_server}}/#/dashboard/10000019`.
- **What you get:** a list of the agents in the analyzed databases, including who signed each agent. You can filter by Agent Name, Trigger Type, Signer, Server Name, Database and Is FocusDB.
- **Fits questions like:** which agents run in our focus databases, and how are they triggered? Which agents run on a particular server? Who signed the agents in this application?

### Code Search (Custom Insights)

- **Where:** Custom Insights › Code Search, `{{idna_server}}/#/codesearch`.
- **What you get:** a search of the source code iDNA collected from the focus databases, as plain text, whole words or a regular expression, with matches listed per code block together with database, design element and matching line. A search expression can be saved as a Custom Insights pattern.
- **Fits questions like:** which of our applications use a particular LotusScript class or function? Where does a particular API call appear in our code? Can I check what a search expression matches before saving it as a pattern? Check [[concepts/shipped-insight-catalog]] first: a shipped insight may already detect the pattern.

### Code Search Results Overview

- **Where:** `{{idna_server}}/metabase/dashboard/10000035`. Not in the Portal menu; it opens in Metabase and depends on your Metabase access.
- **What you get:** the results view of Custom Insights › Code Search, showing for each match the databases and design elements where that code occurs. Its filters are Regular Expression, which holds the search text, and Search Mode (text, whole word or regular expression).
- **Fits questions like:** where do I see the full result list of a code search? In which databases and design elements does this code pattern occur?

### Manage (Custom Insights)

- **Where:** Custom Insights › Manage, `{{idna_server}}/#/custominsights`.
- **What you get:** the place where Rulesets, Insights and Patterns are maintained, one tab each: a ruleset groups insights, and insights are found through patterns, which are code search expressions. The rulesets panagenda ships are read-only and can be changed as a copy; your own can be created, copied, exported and imported, and edits reach the Insights results with the next design analysis run.
- **Fits questions like:** can we add Insights rules for code patterns specific to our organization? Which rulesets ship with iDNA (listed in [[concepts/shipped-insight-catalog]])? Can we adjust a ruleset that panagenda ships?
