---
id: glossary/ntf
title: NTF
type: glossary
status: verified
tags: [domain/notes]
related:
  - concepts/database-template
  - concepts/about-template-inheritance
requires_capability: none
source: CoCo Copilot server/knowledge/foundation_domain_knowledge.md
last_reviewed: 2026-04-17
---

# NTF

**Notes Template Facility.** The `.ntf` file format for HCL Notes database templates. An NTF defines the design (forms, views, agents, code) that other databases inherit from. When the master template is updated, inheriting databases receive the design changes on their next refresh — fixing once in the template propagates to all consumers.

NTFs are typically **not** Focus DBs, so queries against `cqx_data.database` must drop the `is_focus_db` filter to find them. Master templates are identified by `template_acts_as_master IS NOT NULL`.

**See also:** [[concepts/database-template]] for depth, [[concepts/about-template-inheritance]] for the inheritance model.
