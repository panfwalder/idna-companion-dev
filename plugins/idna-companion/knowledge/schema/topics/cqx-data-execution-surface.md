---
id: schema/topics/cqx-data-execution-surface
title: cqx_data as the Public Execution Surface
type: schema
status: verified
tags: [domain/idna, topic/usage, topic/modernization]
related:
  - schema/tables/database
  - schema/tables/replica-set
  - schema/tables/usage-summary-by-database
  - schema/tables/design-insights-summary-notes64bit
  - schema/tables/design-insights-summary-nomadwebmobile
  - schema/topics/department-and-location
  - schema/tables/dbviews-used
requires_capability: none
source: iDNA_Applications_ETL obsidian/code-wiki/architecture/cqx-data.md
last_reviewed: 2026-06-22
---

# cqx_data as the Public Execution Surface

## Purpose

This topic explains the role of `cqx_data` in the broader ETL architecture. The table and topic pages in this wiki describe the public query surface itself; this page adds the architectural context from the ETL/code wiki.

## Why it matters

`cqx_data` is the primary public SQL surface exposed to CQX/DataMiner users. In practice, consultants should prefer `cqx_data` objects over reaching into upstream schemas unless a specific diagnostic question requires lower-level detail.

That is why pages in this wiki focus on `cqx_data.*` tables and views even when the underlying derivation logic lives elsewhere.

## Upstream lineage

The ETL/code wiki describes `cqx_data` as a serving layer built mainly from upstream mart and warehouse schemas:

- `dda_dm` - main modern serving mart
- `idna_dwh` - warehouse-level reference and usage structures
- `idna_dm` - legacy mart data still referenced by some `cqx_data` objects
- `cqx_ctrl` - materialized-view definitions and control metadata

Inference: when a `cqx_data` result looks surprising, the root cause may live upstream, but normal consultant-facing analysis should still start at the `cqx_data` layer.

## Build and refresh model

According to the ETL/code wiki, the `cqx_data` schema is initialized from `cqx_data_ifa/*.pxsql` sources and acts mostly as a live execution surface:

- there is init/build logic for creating the schema objects
- many `cqx_data` objects are views over upstream data
- there is no separate full "refresh the cqx_data schema data" process in the same sense as warehouse or mart table refreshes

That means freshness questions usually trace to upstream refresh state rather than to an independent `cqx_data` load step.

## How to use this context in answers

- Prefer `cqx_data` objects first when proposing consultant SQL.
- Mention upstream lineage only when it helps explain behavior, limitations, or refresh timing.
- Do not overstate lineage at the individual table level unless the ETL/code repo provides direct evidence for that object.

## Sources

- ETL/code wiki architecture page: `C:\Users\fwalder\Documents\panagenda\git\iDNA_Applications_ETL\obsidian\code-wiki\architecture\cqx-data.md`
- ETL/code wiki overview: `C:\Users\fwalder\Documents\panagenda\git\iDNA_Applications_ETL\CLAUDE.md`
