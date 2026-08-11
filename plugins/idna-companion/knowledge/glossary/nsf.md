---
id: glossary/nsf
title: NSF
type: glossary
status: verified
tags: [domain/notes]
related:
  - concepts/database-instance
  - concepts/about-replica-sets
requires_capability: none
source: CoCo Copilot server/knowledge/foundation_domain_knowledge.md
last_reviewed: 2026-04-17
---

# NSF

**Notes Storage Facility.** The `.nsf` file format used by HCL Notes and Domino. Each NSF is a self-contained database holding documents, views, forms, agents, and code (LotusScript, Formula Language, JavaScript, Java, XPages). A single Notes/Domino environment can contain thousands of NSFs distributed across multiple servers.

When a user says "database," they typically mean one NSF — a single database instance. Multiple NSFs with the same Replica ID form a replica set and together represent one application.

**See also:** [[concepts/database-instance]] for depth, [[concepts/about-replica-sets]] for how multiple NSFs aggregate into an application.
