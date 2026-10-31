---
id: howtos/handle-query-timeouts
title: Handle a query that times out - what the signal means, narrow first, DataMiner for long queries
type: howto
status: verified
tags: [domain/idna, audience/consultant, audience/customer]
related:
  - howtos/use-source-code-remediation-mcp-tools
  - howtos/navigate-code
  - schema/tables/code-element
  - schema/topics/cqx-data-execution-surface
  - concepts/data-freshness
requires_capability: none
source: MCP server error mappings, timeout hints and tool descriptions per docs/plans/active/mcp-code-search step T (2026-10-23); client ceilings per the same plan (F-2, T-F4); DataMiner as the route for long queries (product owner, 2026-10-23); the next step as a choice with search_code per step V (2026-10-24); the card carries the result, the timeout signals, the lock wait and the longer-waiting clients per step W (2026-10-24); the answer as a block of its own before the card (a one-paragraph timeout note was folded away with the tool steps in the step-V and step-W eval runs) and the chosen route run as described per step X (2026-10-27); the list rule for the two calls per step Y (2026-10-31)
kbase_sources: [dataminer-query-viewer, dataminer-_index]
last_reviewed: 2026-10-31
---

# Handle a query that times out

## Question

"The query timed out." "It keeps timing out - what now?" "Can this run longer?" "Why does the tool say TIMEOUT?"

## Approach

Every query an MCP tool runs on the iDNA data has a time limit on the server, and the chat client has its own. Most timeouts come from the shape of the request, so narrow it first. A question that really needs a long query runs in DataMiner, iDNA's own SQL screen, which is not bound by the MCP's limit.

## Data needed

None beyond the failed call: its error code, message and hint. For the shapes that narrow code searches see [[howtos/navigate-code]].

## What you see

| Signal | Meaning | Move |
|---|---|---|
| `TIMEOUT` "The query exceeded the server-side statement timeout." | The request ran past the server's limit. The server cancelled the statement; nothing is still running. | The procedure below. Where the error carries a hint, it names what this tool can narrow. |
| `TIMEOUT` "The query waited too long for a table lock." | Another transaction holds a table lock - often the nightly data refresh (the ETL run from 18:30 server time, [[concepts/data-freshness]]). | Wait until the refresh has finished and retry the same request once; a lock wait that repeats outside the refresh goes to the appliance operator. |
| The client's own request timeout (in Cowork: "Request timed out"), with no result from the MCP | The call outlived the client's ceiling. This signal alone does not confirm that the server stopped the statement. | Treat it like a statement timeout, without saying the statement stopped. A longer server limit does not help this client. |
| `TIMEOUT` "The query was cancelled by the client." | The client or the user stopped the call. | Do not retry unasked: if the user cancelled, ask before running anything; otherwise treat it like the client's own timeout. |
| `INTERNAL` "timeout exceeded when trying to connect" | The server is busy: no database connection was free. | Retry once after a short pause; if it happens again, narrow the request. |

A `search_code` refusal for a pattern that matches too many code blocks is not a timeout: the search stops once it has counted too many matching blocks and lists none. Narrow as its hint says.

## The limits

- Every tool's queries on the data have a server-side statement limit of about half a minute. Where the MCP build has `search_code`, `execute_sql` and `search_code` get a longer one, about 50 seconds by default.
- The chat client adds its own ceiling. Cowork stops a tool call after 60 seconds.
- The appliance operator sets the server limits; the MCP server's admin documentation describes them.

## Procedure

1. **Narrow the request or change its shape.** One application instead of the estate, a design-document or code-element key instead of a text search, a more specific pattern, a smaller `top_n` or `limit`. For code text use `search_code` where the build has it; in SQL read `cqx_data.code_source` first and the locations second (two calls, see [[howtos/navigate-code]]); a per-database or per-application list is complete only when the first call returned every matching hash within the row limit. Most timeouts end here.
2. **Use the semantic tool that answers the question,** if there is one - for code text `search_code`, where the build has it (one call, the blocks and their locations). For a statement the user prescribed, offer it rather than run it (Interpretation). Never rebuild a semantic tool's logic in SQL; for lifecycle decisions there is no SQL route at all.
3. **Still too long, and the question needs SQL:** the user can run the query in DataMiner, which is not bound by the MCP's limit. DataMiner opens from the Portal footer ("Open DataMiner") and depends on the user's DataMiner access. Give the SQL with a one-line description of what it shows; the user runs it there with ▶▶ (run all) for the full result - ▶ shows a preview with a row limit - and copies the result back into the conversation if needed. The Companion cannot read DataMiner results itself.
   - The SQL follows the session's rules like any SQL shown here: in a restricted session no per-user method - state what is not available and offer the aggregate alternative first; a question about a named individual stays a clean refusal, with no SQL and no alternative.
   - For a long query the user needs again: a saved DataMiner query can be scheduled on its Options tab for automatic nightly data preparation, so its result is ready without waiting (saving a query needs DataMiner's Editor Mode).
4. **For consultants working with the appliance operator:** on builds with `search_code`, the operator can raise the longer limit of `execute_sql` and `search_code` (`MCP_EXTENDED_STATEMENT_TIMEOUT_MS`; see the MCP server's admin documentation). It helps only with a client whose own tool-call timeout is longer - an unverified path, not exercised against this server: the GitHub Copilot app (a timeout per MCP server, 180 seconds by default) or Claude Code (`MCP_TOOL_TIMEOUT`, or the `timeout` of the server's entry, per its documentation). Other clients have their own setting. Cowork stops at 60 seconds whatever the server allows.

## Interpretation

- A statement or client timeout, or a cancellation, is not "the MCP is unavailable": do not fall back to the wiki's SQL patterns as if the server were down, and do not retry the same request unchanged (the lock-wait and connection rows above are the exceptions).
- During the appliance's nightly data refresh, queries run slower and lock waits happen: wait and retry once after it.
- Answer first - what happened (for each signal as the card's opener below says), what you tried and the hint where the error carries one, as a block of its own: a bold lead line (**What happened**) and a short list - the cause, what ran, the hint - not a single paragraph between tool steps, which a runtime can fold away with them; then, where a choice remains, end with an `AskUserQuestion` (concise prose only where the runtime has none), not a prose list of options: the semantic tool where one answers the question (for code text `search_code`, where the build has it), the narrowed query, or the SQL as written in DataMiner. A choice always remains when the user prescribed the statement; before any choice, when the statement was yours and a semantic tool answers the question, say what you change and run it; once the user has chosen a route, run or hand over that route as described (persona, Interactive follow-up). The card carries the result, since it may be all the user sees while it waits. Its question opens with what happened: for the server's statement timeout, the query ran past the time limit and the server cancelled it; for the client's own timeout, the client stopped waiting, and whether the statement stopped is not confirmed. Each option gives the fact the choice depends on - `search_code`: one call that finds the blocks whose stored code contains the text, with their locations; a candidate search with no language filter, in the Focus applications unless the user asks for all databases (say so when the statement covered the whole estate); the narrowed query: two calls, a list complete only when the first returned every matching hash within the row limit; DataMiner: runs the SQL as written and is not bound by this limit.

## Variations

- "Can the Companion read my DataMiner results?" No: the MCP reads `cqx_data` only. Paste the result, or read it in DataMiner or Metabase.
- A semantic tool times out on one application: narrow its options (a smaller `top_n`, one ruleset or window); if that does not help, say so - the limit is set on the server, not per call.
