---
id: howtos/list-master-templates
title: List master templates in the environment
type: howto
status: verified
tags: [domain/idna, topic/modernization]
related:
  - concepts/database-template
  - concepts/about-template-inheritance
  - concepts/focus-db
  - schema/tables/database
  - schema/topics/template-inheritance
requires_capability: none
source: CoCo Copilot server/knowledge/foundation_domain_knowledge.md
last_reviewed: 2026-04-17
---

# List master templates in the environment

## Question

"Which master templates (NTF files serving as design parents) exist in this environment?"

## Approach

Query `cqx_data.database` for rows where `template_acts_as_master IS NOT NULL`. Important: **do not filter `is_focus_db`** — templates are typically not Focus DBs and would be hidden by the default scope.

## Data needed

- [[schema/tables/database]] — `template_acts_as_master`, `title`, `server_name`, `file_name`.
- Background: [[schema/topics/template-inheritance]].

## Procedure

1. State the scope change to the user: "Dropping the Focus filter — templates are not Focus DBs." See [[concepts/focus-db]] for why.

2. Run the query.

   Intent: list every database instance that acts as a master template.

   ```sql
   SELECT title, template_acts_as_master, server_name, file_name
   FROM cqx_data.database
   WHERE template_acts_as_master IS NOT NULL
   ORDER BY template_acts_as_master, server_name
   ```

3. For each template, the consultant often wants to know how many applications inherit from it. Follow up with [[schema/tables/replica-set]]:

   Intent: count applications inheriting from a specific template name.

   ```sql
   SELECT COUNT(*) FROM cqx_data.replica_set
   WHERE '<template_name>' = ANY (templates_inherits_from)
     AND is_focus_db
   ```

## Interpretation

- `template_acts_as_master` is the template's declared name — this is the value that appears in `templates_inherits_from` on inheriting databases.
- Multiple instances of the same master template can exist (e.g. copies on different servers). Expect duplicates; the template name is what matters for inheritance, not the file path.
- A template with zero inheriting applications is a candidate for cleanup — it's design overhead with no consumers.

## Variations

- **Standard HCL templates specifically** — filter by known names (StdR4Mail, StdDocLib, StdR4Disc, etc.) or cross-reference with applications where `design_is_similar_to_standard_template = true`.
- **Templates on a specific server** — add `WHERE server_name = '<server>'`.
- **Usage on the template itself** — join to [[schema/tables/usage-summary-by-database]] via `database_id`. Expect low usage; templates are rarely opened by end users.

## Sources

CoCo Copilot foundation knowledge.
