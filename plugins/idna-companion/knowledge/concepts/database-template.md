---
id: concepts/database-template
title: Database Template (NTF)
type: concept
status: verified
tags: [domain/notes, domain/domino, topic/modernization]
related:
  - concepts/database-instance
  - concepts/about-template-inheritance
  - glossary/ntf
  - howtos/list-master-templates
  - schema/tables/database
requires_capability: none
source: CoCo Copilot server/knowledge/foundation_domain_knowledge.md
last_reviewed: 2026-07-24
---

# Database Template (NTF)

## What it is

A template file (`.ntf`) that defines database design. Databases inheriting from a template receive design changes when the template is updated — fixing one template propagates the fix to every database that inherits from it.

Also called a **Master Template**. NTF stands for **Notes Template Facility**.

## Why it matters for consulting

Templates are the highest-leverage fix path in modernization work. A single template update can resolve findings across hundreds of inheriting databases. The modernization vector and plan precompute exactly this leverage ([[concepts/about-modernization-vector]], [[concepts/about-modernization-plan]]); the legacy A/B/C/D framing lives on [[concepts/remediation-category]].

## iDNA data

Templates are usually **not** Focus DBs. To list master templates, query `cqx_data.database` **without** the `is_focus_db` filter:

```sql
SELECT title, template_acts_as_master, server_name, file_name
FROM cqx_data.database
WHERE template_acts_as_master IS NOT NULL
```

Related howto: [[howtos/list-master-templates]].

## Related glossary

- [[glossary/ntf]]

## Sources

CoCo Copilot foundation knowledge.
