# Knowledge Wiki Index

Catalog of all wiki pages, grouped by type. Updated on every ingest.

## Concepts

- [[concepts/notes-domino-environment]] - HCL Notes/Domino environment
- [[concepts/database-instance]] - Database instance (NSF)
- [[concepts/database-template]] - Database template (NTF)
- [[concepts/about-replica-sets]] - Replica set
- [[concepts/replica-id]] - Replica ID
- [[concepts/focus-db]] - Focus DB / Focus Replica Set
- [[concepts/user-access-days]] - User Access Days (UAD)
- [[concepts/about-template-inheritance]] - Template inheritance
- [[concepts/remediation-classification]] - Remediation Classification (application-level)
- [[concepts/usage-tier]] - Usage Tier (qualitative usage grade)
- [[concepts/lifecycle-vector]] - Lifecycle Vector (ETL-computed lifecycle bucket: `direction/disposition/reason`)
- [[concepts/about-modernization-vector]] - Modernization Vector — READINESS: "what apps will break on 64-bit clients", "which apps aren't Nomad-ready" (per-target readiness split + fix routing)
- [[concepts/about-modernization-plan]] - Modernization Plan — "where do we start", "what pays off most", session worksheets (session queue + block worklist)
- [[concepts/about-template-lineage]] - Template Lineage — detached forks of standard templates, rebase vs upgrade, "what did we customize / what survives the update", ready with notes (HCL-owned findings), the step-by-step session procedure
- [[concepts/design-families]] - Design Families (similarity components; unify-under-one-master candidates)
- [[concepts/database-complexity]] - Database Complexity (weighted design/code score, how to read it)
- [[concepts/design-similarity]] - Design Similarity (clusters, template candidates, complexity corrector)
- [[concepts/compatibility-ruleset-routing]] - Bind the target ruleset before using the shared remediation workflow
- [[concepts/notes-64bit-remediation-profile]] - Bound workflow profile for Notes 64-bit remediation
- [[concepts/nomad-web-mobile-remediation-profile]] - Bound workflow profile for Nomad Web/Mobile remediation
- [[concepts/notes-64bit-findings-taxonomy]] - Compact classification of Notes 64-bit finding types
- [[concepts/nomad-web-mobile-findings-taxonomy]] - Compact classification of Nomad Web/Mobile finding types
- [[concepts/code-block]] - Code block (identical code by content hash; when it is shared through a template and when it is a copy)
- [[concepts/remediation-category]] - Remediation Category (LEGACY A/B/C/D scheme → vector-route mapping; current vocabulary = the modernization vector routes)
- [[concepts/notes-64bit-compatibility]] - Notes 64-bit compatibility
- [[concepts/nomad-web-mobile-compatibility]] - Nomad Web/Mobile compatibility
- [[concepts/regulatory-usage-constraints]] - Strict vs permissive usage analysis: AI Act, works council, retention
- [[concepts/usage-time-windows]] - Canonical buckets vs custom windows: the three-rung ladder, explicit-window rule, comparability guard

## Schema

### Tables

- [[schema/tables/replica-set]] - `cqx_data.replica_set` (application-level)
- [[schema/tables/replica-set-summary]] - `cqx_data.replica_set_summary` (compact KPI rollup: complexity/usage/remediation labels + indices)
- [[schema/tables/database]] - `cqx_data.database` (per-instance, master templates)
- [[schema/tables/design-summary-by-replica-set]] - `cqx_data.design_summary_by_replica_set` (complexity + per-language code volumes)
- [[schema/tables/design-insight-overview-bydb]] - `cqx_data.design_insight_overview_bydb` (per-finding by DB; standard-template detection)
- [[schema/tables/usage-summary-by-database]] - `cqx_data.usage_summary_by_database`
- [[schema/tables/dbviews-used]] - `cqx_data.dbviews_used` (used views per DB instance; view-index usage)
- [[schema/tables/design-insights-summary-notes64bit]] - `cqx_data.design_insights_summary_notes64bit`
- [[schema/tables/design-insights-summary-nomadwebmobile]] - `cqx_data.design_insights_summary_nomadwebmobile`
- [[schema/tables/design-insight-overview-byrepset]] - `cqx_data.design_insight_overview_byrepset` (finding detail, one row per occurrence; masked rows on trial)
- [[schema/tables/code-source]] - `cqx_data.code_source` (code block source text)
- [[schema/tables/department]] - `cqx_data.department` (org hierarchy)
- [[schema/tables/location]] - `cqx_data.location` (location hierarchy)
- [[schema/tables/usage-cube-repset-vs-department]] - app x department cube
- [[schema/tables/usage-cube-repset-vs-location]] - app x location cube
- [[schema/tables/usage-cube-repset-vs-user]] - app x user cube (permissive only)
- [[schema/tables/repset-activity-summary-per-user]] - per-user activity summary by time period (permissive only)
- [[schema/tables/usage-cube-department-summary]] - department usage summary
- [[schema/tables/usage-cube-department-activity-history]] - department weekly activity trend
- [[schema/tables/usage-cube-location-summary]] - location usage summary
- [[schema/tables/usage-cube-location-activity-history]] - location weekly activity trend
- [[schema/tables/usage-sessions-by-department]] - department session detail
- [[schema/tables/usage-sessions-by-location]] - location session detail
- [[schema/tables/usage-sessions]] - raw session detail per (database, day, user, server) (permissive only)
- [[schema/tables/usage-cube-repset-activity-history]] - per-app weekly activity trend (custom time windows)
- [[schema/tables/usage-cube-weekly-summary]] - estate-level weekly usage summary
- [[schema/tables/user-activity-summary]] - person-grain activity + directory link, person_id/username resolution (permissive only)
- [[schema/tables/usage-cube-raw]] - unfiltered usage cube, 20 grouping sets, GROUPING-SET WARNING (permissive only)
- [[schema/tables/usage-cube-raw-no-summaries]] - usage cube leaf grain, session metrics only (permissive only)
- [[schema/tables/usage-cube-repset-vs-department-activity-history]] - app x department weekly activity (custom windows, incl. UAD; -1 = unattributed)
- [[schema/tables/usage-cube-repset-vs-location-activity-history]] - app x location weekly activity (custom windows, incl. UAD; -1 = unattributed)
- [[schema/tables/database-content-retention-summary]] - `cqx_data.database_content_retention_summary` (per-DB content age + retention status)
- [[schema/tables/database-content-retention]] - `cqx_data.database_content_retention` (monthly modification buckets, retention-window flag)
- [[schema/tables/modernization-vector]] - `cqx_data.modernization_vector` (per app x ruleset readiness; GRAIN WARNING)
- [[schema/tables/modernization-plan]] - `cqx_data.modernization_plan` (session queue, ranked by apps cleared)
- [[schema/tables/modernization-worklist]] - `cqx_data.modernization_worklist` (+ `_by_session` dossier; block-grain fixes, reach-first)
- [[schema/tables/template-ancestry]] - `cqx_data.template_ancestry` (which standard template a master was forked from; similarity, family, newest version)
- [[schema/tables/modernization-lineage-dossier]] - `cqx_data.modernization_lineage_dossier` (what we customized vs the ancestor; which findings survive the template update)
- [[schema/tables/modernization-session-steps]] - `cqx_data.modernization_session_steps` (ordered step-by-step procedure per plan session; ORDER BY step_rank)
- [[schema/tables/code-block-hcl-origin]] - `cqx_data.code_block_hcl_origin` (is this block current HCL template code, from which family and versions; join by code_hash)

### Topics

- [[schema/topics/cqx-data-execution-surface]] - `cqx_data` as the public query surface
- [[schema/topics/template-inheritance]] - Template inheritance across tables
- [[schema/topics/complexity]] - Complexity columns across tables (computed family, volumes, manual field)
- [[schema/topics/similarity]] - Design similarity columns across tables (clusters, standard-template detection)
- [[schema/topics/trial-license-masking]] - What trial licenses mask across cqx_data (pseudonyms, detail_masked, `Master n` aliases), the MCP trial flag, finding rows on trial, and the occurrence / finding / block units
- [[schema/topics/department-and-location]] - Department/location table families and pairing

## Playbooks

### Application Lifecycle

- [[playbooks/application-lifecycle/lifecycle-routing]] - Routing map for lifecycle / consolidation / sunset work
- [[playbooks/application-lifecycle/sunset-funnel]] - Sunset funnel — delete or archive
- [[playbooks/application-lifecycle/keep-funnel]] - Keep funnel — modernize or qualify
- [[playbooks/application-lifecycle/stakeholder-cohort]] - Stakeholder cohort overlay (who depends on this app)

### Source-Code Remediation

- [[playbooks/source-code-remediation/remediation-routing]] - Routing map for compatibility-oriented source-code remediation work
- [[playbooks/source-code-remediation/assess-exposure]] - Readiness split vs. exposure volume: which lens answers "how ready / what breaks" vs. "how big / how severe"; the trial exception when the finding source is unavailable
- [[playbooks/source-code-remediation/triage-working-set]] - Business overlays + lifecycle scope cuts on the precomputed plan
- [[playbooks/source-code-remediation/classify-template-paths]] - Template-based remediation paths (vector routes: remediate_master / upgrade_standard / investigate)
- [[playbooks/source-code-remediation/classify-shared-code-paths]] - Shared-code (fix-once) vs. unique remediation via the worklist
- [[playbooks/source-code-remediation/analyze-shared-vs-solo-blocks]] - "Which fixes do we author once and reapply?" — the shared-vs-solo analysis workflow (tool chain, two reach scopes, drift)
- [[playbooks/source-code-remediation/build-remediation-plan]] - Sequence the precomputed plan into a roadmap (three tiers + business overlays)

## Glossary

- [[glossary/nsf]] - NSF
- [[glossary/ntf]] - NTF
- [[glossary/uad]] - UAD
- [[glossary/business-critical]] - Business Critical (remediation_classification value)

## Howtos

- [[howtos/find-most-used-applications]] - Find the most used applications
- [[howtos/analyze-usage-by-department]] - Analyze application usage by department
- [[howtos/list-master-templates]] - List master templates in the environment
- [[howtos/use-source-code-remediation-mcp-tools]] - Route remediation questions to the MCP tool surface
- [[howtos/use-application-lifecycle-mcp-tools]] - Route lifecycle / consolidation questions to the MCP tool surface
- [[howtos/attribute-usage-aggregate-only]] - Strict-mode attribution: department/location rank without per-user detail
- [[howtos/identify-content-creators-and-consumers]] - Permissive-mode attribution: per-user heavy-user detection and content-creator vs consumer split
- [[howtos/query-modernization-views]] - Query mechanics for the modernization views (ruleset filter, prefix search, items, reach sources)
- [[howtos/analyze-usage-over-custom-window]] - Custom-window usage via the usage_over_window tool + weekly-table SQL (restricted-safe)
- [[howtos/analyze-raw-user-activity]] - Per-user activity at daily grain from any angle, via user_activity_over_window + the raw pattern (permissive only)

## Support documents

| File | Purpose |
|---|---|
| `_rules.md` | Wiki structure rules (page types, frontmatter, sensitivity, tags, links, IDs). Ships at runtime. |
| `_maintenance.md` | Dev-time ingest, lint, and bootstrap workflows. Excluded from dist. |
