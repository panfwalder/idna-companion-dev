---
id: concepts/code-block
title: Code Block
type: concept
status: verified
tags: [domain/idna, topic/modernization]
related:
  - concepts/remediation-category
  - schema/tables/code-source
  - schema/tables/design-insight-overview-byrepset
  - playbooks/source-code-remediation/classify-shared-code-paths
  - schema/topics/template-inheritance
  - schema/tables/code-block-hcl-origin
  - schema/tables/modernization-worklist
requires_capability: none
source: CoCo_Playbook_64bit_Migration_Consulting.docx + CoCo seed material; the shared-vs-copied rule from plan code-block-eval-fixes (D-7, the external audit's EX-3 wording; eval gq-08, 2026-10-02) and the get_code_block notes of the 2026-10 MCP build
last_reviewed: 2026-10-02
---

# Code Block

## What it is

A unit of source code identified by iDNA via a content hash. The same normalized block may appear in many different applications.

## Why it matters

The docx treats the code-block lens as one of the main efficiency insights of the engagement: fixing one high-impact block can resolve findings across many applications at once.

## Shared or copied?

The same `code_hash_id` in several apps proves identical normalized code; it does not by itself prove how the code arrived there or that the apps share one maintained source. Bind the compatibility target first. A template path is established for particular carriers only when the target-bound worklist or `modernization_for_app` shows that an estate-operated master contains this exact block and covers those carriers (for example through `n_masters_containing`, `master_targets`, and per-app coverage). `templates_inherits_from` and similarity are relationship context, not proof that the template contains the block. Without exact-block coverage, treat the kept occurrences as copies for planning: fix them per copy, or author once in the session named by `scope` and `authored_in`. Sunset carriers are retirement candidates, not fix targets. Current HCL code is a separate no-authoring case: call it "maintained by HCL" only for a covered `persists_hcl_owned` lineage occurrence; call an `hcl_current` occurrence without that verdict a verbatim copy.

`get_code_block`'s provenance note says which case the block-level data supports (from the 2026-10 MCP build): "copies" when no carrier reports an inherited or similar template, "not proof of one shared source" otherwise. Its `downstream_templates` counts inherited OR similar template names reported by the carriers, so a value above 0 is not proof either. Its `hcl_origin` names the HCL template family and versions when the exact code is shipped HCL code ([[schema/tables/code-block-hcl-origin]]).

## iDNA data

- finding-level references: [[schema/tables/design-insight-overview-byrepset]]
- normalized source lookup: [[schema/tables/code-source]]

## Playbook use

The shared-code and unique-code analysis is handled in [[playbooks/source-code-remediation/classify-shared-code-paths]].

## Sources

64-bit migration consulting docx and CoCo seed material; the shared-vs-copied rule from the 2026-10-02 eval fixes.
