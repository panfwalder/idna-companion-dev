# Companion Knowledge Wiki Rules

This document defines how the Companion's knowledge wiki is **structured** — page types, frontmatter, sensitivity, tags, links, IDs.

It is intended for both runtime (the deployed Companion reading the wiki) and dev-time (authors maintaining the wiki). It contains **no dev-time workflow** (ingest, lint, bootstrap). Those live in `_maintenance.md` next to this file (dev repo only; excluded from dist).

---

## Optimization target

The wiki is part of an AI skill. Optimize structure primarily for **agent retrieval efficiency**, not for human reading flow.

That means:

- prefer pages that match concrete consultant/customer intents
- prefer compact pages with clear scope boundaries over long narrative documents
- organize playbook knowledge around what the agent needs to do next, what evidence it needs, and which wiki nodes should be retrieved together
- treat human-friendly sequencing as secondary when it conflicts with retrieval precision

## Location

The wiki lives at `plugin-source/knowledge/` in the dev repo and ships unchanged inside the deployed plugin. **Runtime is read-only.** The deployed Companion does not modify wiki files.

## Page types (five)

Legacy wording below may still mention phase-oriented playbooks from the CoCo scaffold. The governing rule is: playbook pages are task-oriented retrieval units, not mandatory phase slices.

| Type | Purpose |
|---|---|
| `concept` | A Notes/Domino or iDNA idea explained in depth. Reusable across engagements. |
| `schema` | A CQX table/view/column, or a cross-cutting schema topic. |
| `playbook` | A project-flow page — overview + phase pages for a structured engagement (e.g., 64-bit migration). |
| `glossary` | A short terminology lookup (<200 words, self-contained). |
| `howto` | A reusable question-recipe combining concepts + schema to answer a consultant intent. |

**Concept vs. glossary.** If <200 words and self-contained → `glossary`. If structure, examples, or a related-page map is required → `concept`. A glossary entry that grows gets promoted to `concept` (leave a stub redirect behind).

**Playbooks and howtos.** A playbook page requiring a reusable analysis *links out* to a `howto` rather than duplicating the analysis inline. Howtos are owned once, referenced many times.

Playbook structure should follow **agent task boundaries** and **decision boundaries**, not inherited source-document chaptering unless that chaptering is clearly the best retrieval structure.

Good playbook pages are keyed to things like:

- determining scope
- sizing exposure
- deciding whether an application should be remediated at all
- identifying the highest-leverage remediation path
- preparing a worklist or project plan

Avoid preserving a phase split only because the source or a legacy scaffold used one.

## Directory layout

```
knowledge/
├── _rules.md              # this file (structure rules — ships at runtime)
├── _maintenance.md        # dev-time ingest / lint / bootstrap workflow (excluded from dist)
├── _templates/            # page-type templates (excluded from dist)
├── index.md               # content catalog
├── concepts/
├── schema/
│   ├── tables/
│   └── topics/
├── playbooks/
│   └── <playbook-name>/
├── glossary/
└── howtos/
```

Dev-time bookkeeping artifacts (`todo.md`, `log.md`) live at `docs/plugin/` in the dev repo, not in `knowledge/`.

## Frontmatter schema

All pages include:

```yaml
id:                   # stable, path-based or semantic; never changes once assigned
title:                # human-readable
type:                 # concept | schema | playbook | glossary | howto
status:               # draft | verified | needs-update | deprecated
tags: []              # controlled, namespaced (see Tag namespaces)
related: []           # stable IDs of related pages; primary graph mechanism
requires_capability:  # access capability needed to view this page; default: none
source:               # optional: origin of the knowledge (free text or citation)
last_reviewed:        # YYYY-MM-DD
```

**Rules:**
- `id` equals the page's path relative to `knowledge/` without `.md`. If the file moves, `id` moves with it. See [ID strategy](#id-strategy) for the rule and [`_maintenance.md`](_maintenance.md) Part 2 for the page-move procedure.
- `status: draft` until a primary authoritative source has shaped the page. CoCo-derived pages start `draft`.
- `related:` lists the pages this one depends on or extends. Obsidian includes frontmatter links in the graph.
- `source:` free text; cite the origin when knowledge came from a specific document.

## Sensitivity (`requires_capability`)

The `requires_capability` field declares which access capability a *reader* must have for the page to be visible at runtime. The vocabulary is **mapped/aligned with** (not literally identical to) the MCP role design in `docs/decisions/_artifacts/graph-concept-counter.md` and ADR `docs/decisions/0006-license-flag-model-and-introspection-runtime.md`. Three distinct sets of names exist deliberately — wiki capability values, MCP envelope `access_profile` values, and Postgres role names — because each layer talks about a different concern (what kind of content this is, what kind of session is running, who has SELECT on what).

| Layer | Values |
|---|---|
| Wiki `requires_capability` | `none`, `mcp_unrestricted` |
| MCP envelope `access_profile` | `restricted`, `privileged`, `none` |
| Postgres group roles | `cqx_role_mcp_restricted`, `cqx_role_mcp_privileged` |

Allowed values:

| Value | Visible when |
|---|---|
| `none` (default) | Always — visible to both restricted and permissive MCP modes |
| `mcp_unrestricted` | Visible only when the MCP login user is a member of `cqx_role_mcp_privileged` (permissive mode) |

The capability name (`mcp_unrestricted`) and the Postgres role name (`cqx_role_mcp_privileged`) are deliberately different: the wiki layer talks about *what kind of content this is*, while the data layer talks about *which role grants access to what*. They map 1:1 today but are kept conceptually distinct so future divergence (e.g., a third profile, or a wiki page that requires multiple capabilities) can be handled without renaming.

**Authoring rule.** Pages that *describe* policy or regulatory constraints stay `requires_capability: none` even when they discuss privileged data — explaining the rules belongs in the base profile. Only pages that *teach analysis or recipes* against privileged data carry a higher value.

**Mixed pages get split.** A playbook that mostly works in `estate_safe` but has a section on per-user attribution is split: the privileged content moves to a separate howto with the appropriate `requires_capability`, linked from the playbook via `related:`. The playbook stays `requires_capability: none`. This matches the existing playbook→howto delegation rule.

**One value per page.** A page does not list multiple required capabilities. If you would need to, split the content.

**Runtime filter.** The `idna-wikilookup` skill consults the active profile's enabled capabilities (via the MCP's `get_idna_session()`) on session start and filters retrieval by `requires_capability`. The same wiki bytes ship to every tenant; the runtime filter is what makes strict and permissive tenants experience different content. See `plugin-source/skills/idna-wikilookup/SKILL.md`.

## Tag namespaces

Controlled tags only. Namespaces:

- `domain/...` — e.g., `domain/notes`, `domain/domino`, `domain/idna`, `domain/panagenda`
- `topic/...` — e.g., `topic/64bit`, `topic/nomad`, `topic/consolidation`, `topic/usage`, `topic/modernization`, `topic/regulatory`, `topic/content-age`
- `audience/...` — e.g., `audience/consultant`, `audience/customer`

Do not use `type/...` or `status/...` tags — those are dedicated frontmatter fields.

## Link conventions

- **Primary graph mechanism: `related:` frontmatter.** Every page lists its structural neighbors there.
- **Inline `[[wiki-links]]`** only where the reference is naturally part of the sentence — e.g., a playbook step that says *"Assess usage per application — see [[howtos/find-most-used-applications]]."* Wiki-link target must be the page's `id:` (i.e., the path relative to `knowledge/` without `.md`).
- **External URLs:** standard markdown `[text](https://...)`.
- **Never** use standard markdown links for inter-wiki references — they don't create graph edges.
- **iDNA host URLs use the `{{idna_server}}` placeholder.** Three URL shapes appear:
  - **Metabase dashboards** — `{{idna_server}}/metabase/dashboard/<id>` (e.g. `10000017`).
  - **iDNA advanced components** — `{{idna_server}}/#/<route>` (e.g. `/#/catalog` for the Database Catalog, `/#/masterlistreplicasets` for the Replica Set Catalog).
  - **Per-DB-instance details** — `{{idna_server}}/#/replicadetails/<database_id>`. Detailed view of virtually all collected information for one DB instance. Use as a drill-down whenever analysis lands on a specific `database_id` (e.g. archival-instance pick, replication-divergence investigation, per-instance content-age review). **No replica-set equivalent exists today.**

  Do not assume every iDNA link is a Metabase dashboard.

## ID strategy

**An `id:` is the file's path relative to `knowledge/`, without the `.md` extension.** Resolution is closed-form: `file = knowledge/<id>.md`. No mapping table; no implicit pluralization rules.

Examples:

| File | `id:` |
|---|---|
| `concepts/replica-id.md` | `concepts/replica-id` |
| `schema/tables/database.md` | `schema/tables/database` |
| `schema/topics/department-and-location.md` | `schema/topics/department-and-location` |
| `playbooks/source-code-remediation/assess-exposure.md` | `playbooks/source-code-remediation/assess-exposure` |
| `glossary/uad.md` | `glossary/uad` |
| `howtos/find-most-used-applications.md` | `howtos/find-most-used-applications` |

For playbooks, prefer task-oriented slugs (`assess-exposure`, `build-remediation-plan`) over generic ones (`overview`). The `id:` then naturally reads as `playbooks/<name>/<task-slug>`.

Slugs use lowercase kebab-case.

**Why keep `id:` as a separate frontmatter field when it equals the path?** Two reasons: (a) page-internal grep'ability — a contributor reading any page sees its canonical id at the top; (b) lint self-check — `scripts/wiki-lint.py` asserts `id:` equals the file's path, catching accidental moves where the author forgot to update the frontmatter.

**Page moves.** When a page moves, the `id:` moves with it. See [`_maintenance.md`](_maintenance.md) Part 2 for the page-move procedure (which also covers updating inbound references and current-truth references outside the wiki).

**Concept/schema slug collisions (`about-` prefix rule).** When a `concepts/<slug>.md` would share a basename with a `schema/tables/<slug>.md` or `schema/topics/<slug>.md`, the concept page gets an `about-` prefix to remove the basename collision (e.g. `concepts/about-replica-sets.md` next to `schema/tables/replica-set.md`). The schema page owns the natural slug; the concept page qualifies. This is defense-in-depth against any tooling that resolves links by basename, even though slash-form IDs already disambiguate by path.
