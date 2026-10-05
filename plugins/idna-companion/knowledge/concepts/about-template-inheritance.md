---
id: concepts/about-template-inheritance
title: Template Inheritance
type: concept
status: verified
tags: [domain/notes, domain/domino, topic/modernization]
related:
  - concepts/database-template
  - concepts/remediation-category
  - concepts/design-similarity
  - schema/tables/replica-set
  - schema/tables/database
  - schema/topics/template-inheritance
  - schema/topics/similarity
requires_capability: none
source: CoCo Copilot server/knowledge/foundation_domain_knowledge.md; the declared-link gloss per code-block-eval-fixes Part E / PE-4 (option b) and mcp-trial-awareness EX-11, 2026-10-05
last_reviewed: 2026-10-05
---

# Template Inheritance

## What it is

Databases in Notes/Domino can inherit their design from a **master template**. When the template is updated and the databases are refreshed, the design change propagates automatically. Inheritance is the mechanism that makes fix-once-propagate-many possible.

iDNA identifies two states of inheritance:

- **Clean association** — an explicit template link is set and a master with that name is operated in the estate. When design refresh runs, it can update elements still linked to that template and not protected from refresh; the declared link alone is not a block-coverage verdict.
- **Broken association** — the explicit link is missing (often because a developer disconnected it to prevent accidental overwrites), or it names a master the estate does not operate, but iDNA's similarity analysis still detects the relationship.

## Why it matters

Clean associations are the lowest-effort remediation path — the modernization vector routes them to `template/remediate_master`, `template/upgrade_standard`, or — for a master that is itself a detached fork of a standard template — `template/rebase_ancestor` ([[concepts/about-modernization-vector]], [[concepts/about-template-lineage]]). Broken associations carry the same design lineage but require validation before a fix can propagate — the vector routes them to `template/investigate_inheritance/*`. (Legacy vocabulary: Category A/B — mapping on [[concepts/remediation-category]].)

## Columns involved

At the **replica-set level** (see [[schema/tables/replica-set]]) these columns are arrays — a single replica set can have instances inheriting from different templates:

- `templates_inherits_from` (text[]) — the declared template link; a clean association only when an estate-operated master carries the name.
- `most_similar_template_name_newest_design` (text[]) — iDNA-detected match when explicit link is missing (broken association).
- `most_similar_template_similarity_newest_design` (numeric) — similarity score; higher is stronger.
- `design_is_similar_to_standard_template` (boolean) — quick flag for HCL standard template derivation.

At the **database-instance level** (see [[schema/tables/database]]) the equivalent column is a single value:

- `template_inherits_from` (text).

The array-vs-singleton distinction reflects reality: one application (replica set) can span instances that inherit from different templates.

## Deeper schema topic

See [[schema/topics/template-inheritance]] for column-level detail and the similarity-drilldown table reference.

## Sources

CoCo Copilot foundation knowledge (Template Inheritance section).
