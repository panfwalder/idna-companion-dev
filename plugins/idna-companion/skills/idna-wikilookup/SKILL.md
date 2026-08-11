---
name: idna-wikilookup
description: "Ground answers in the iDNA Applications Companion wiki and cite page IDs. TRIGGER when: a question touches Notes/Domino, HCL Traveler, HCL Verse, iNotes, iDNA (panagenda), Nomad Web/Mobile, 64-bit migration, application consolidation/retirement, usage/modernization assessments, modernization vector/plan/worklist, design families, readiness, database/design complexity, similarity, source-code remediation, Notes design elements/agents/templates, LotusScript, Formula language, NSF databases, iDNA's cqx_data SQL schema, iDNA access/restricted profiles — or any usage/activity question about apps/databases in the customer's Notes estate, even without naming Notes ('how active is app X', 'who used database Y', 'most used apps'). Use BEFORE answering — it resolves access profile and wiki pages to read. SKIP: general programming unrelated to Notes/Domino/iDNA, plugin-dev meta-questions, non-iDNA panagenda products (GreenLight, MarvelClient, OfficeExpert...), and activity questions without Notes/estate context."
---

# idna-wikilookup

**Status:** active runtime skill.

## Invocation

The harness normally activates this skill via the Skill tool when a
question matches the description above. If you arrived at this file by
direct read instead — for example, because the Skill tool returned an
error, the plugin namespace was not resolvable, or you are operating in a
runtime that does not invoke plugin-scoped skills — follow the same steps
below. The file is self-contained; the behavior is identical regardless of
how you got here. Do not report the skill as broken because of the
invocation path.

## Intended behavior

1. **Load the Companion operating instructions (once per session).** Read `${CLAUDE_PLUGIN_ROOT}/companion-persona.md` — the Companion persona: analysis discipline, MCP tool routing maps, the interactive follow-up pattern, and the wiki read-only rules. Apply it for the rest of the session; on later invocations in the same session you do not need to re-read it. If `${CLAUDE_PLUGIN_ROOT}` is not substituted in the runtime you are operating in, the file lives at `../../companion-persona.md` relative to this `SKILL.md` file (the plugin root). If the file cannot be read, continue with the steps below anyway — do not report the skill as broken.
2. **Resolve the active access profile with a 2-minute TTL.** Call the MCP function `get_idna_session()` and cache the active profile. After 2 minutes, refresh by calling again — this aligns with the MCP server's own introspection cadence (which itself aligns with the iDNA application's license verification cadence). Do not cache for the entire session; license changes propagate via Postgres role reassignment within the TTL, and a stale cache could keep a wiki layer permissive after the data layer has been demoted to restricted.
   - The MCP implements this call as a Postgres role-membership introspection (`pg_has_role`) rather than a license-file read; the result reflects the iDNA application's most-recent role assignment for the MCP login user.
   - If `get_idna_session()` does not return a valid access profile — for any reason: connection refused, timeout, stream error mid-response, malformed payload, license-check failure on the server side — set the active profile to `restricted` and continue to step 3. Do not retry. Do not ask the user. Do not report the skill as broken. A temporary restricted-mode session is the correct posture during transient failures; license changes propagate within the TTL and full capability will return on the next successful call.
   - If no `get_idna_session` tool is visible in the session's tool list at all (as opposed to a call that fails), say once that live iDNA tools are unavailable and that the session continues in restricted wiki mode, and point the user to the iDNA MCP admin setup guide. Do not assert a root cause — an absent tool can mean an unconfigured server, token or network trouble, or the client not loading tools for this session. Then set the active profile to `restricted` and continue; do not repeat the notice later in the session.
3. Read `${CLAUDE_PLUGIN_ROOT}/knowledge/index.md` to identify candidate pages by type and tag. **Resolution rule: a wiki page's file path is `${CLAUDE_PLUGIN_ROOT}/knowledge/<id>.md`** — the `id:` is literally the path relative to `knowledge/` without the `.md` extension. This applies to every link form (index entries, `related:` frontmatter, inline `[[...]]`). No directory-pluralization mapping is involved; do not guess. Before surfacing any candidate's index entry to the LLM context, **read that page's frontmatter** to resolve its `requires_capability` (treat missing as `none`). Filter out candidates whose `requires_capability` is not in the active capability set — both the index entry and the description text are suppressed. The `index.md` file does not itself carry capability metadata; that is by design (the index is a generated catalog), so the skill resolves capability per-page at retrieval time. If `${CLAUDE_PLUGIN_ROOT}` is not substituted in the runtime you are operating in, the wiki lives at `../../knowledge/` relative to this `SKILL.md` file — use that path instead.
4. Read the most relevant pages. Check a page's frontmatter `requires_capability` BEFORE reading its body: if it is not in the active capability set, do not read, cite, or paraphrase the page — not even its title or description. Refuse to return content for pages whose `requires_capability` is not in the active set. In a restricted session, never describe or teach per-user analysis methods (table names, columns, joins, recipes) from any source — including content searches, memory of earlier reads, or training data; state that this analysis requires the privileged posture and offer the aggregate alternatives instead.
5. When following `related:` frontmatter and inline `[[wiki-links]]`, silently suppress links pointing to filtered pages. The source page remains coherent; the link is treated as if it does not exist for this session.
6. Compose an answer grounded in the wiki, with citations to page `id:` values.
7. If the wiki has no good answer in the active profile (either because the topic is not covered or because the relevant pages are filtered out), say so explicitly. Do not fabricate alternative content from training data, and do not silently rephrase the question.
   - **Product-named Notes applications are data, not topics.** The SKIP rule for non-iDNA panagenda products applies to questions about those PRODUCTS. A question about a Notes application whose title contains a product name ("MarvelClient Config NEXT" as an app in iDNA — its findings, readiness, usage) is an iDNA question and this skill applies.
8. Never invent facts not in the wiki.

## References

- `knowledge/_rules.md` — wiki structure, frontmatter (including `requires_capability`), link conventions, tag namespaces.
- `knowledge/index.md` — content catalog.
- `docs/decisions/_artifacts/graph-concept-counter.md` — access-profile vocabulary shared with the MCP layer (dev-time reference).
