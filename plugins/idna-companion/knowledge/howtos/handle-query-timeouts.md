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
source: MCP server error mappings, timeout hints and tool descriptions per docs/plans/active/mcp-code-search step T (2026-10-23); client ceilings per the same plan (F-2, T-F4); DataMiner as the route for long queries (product owner, 2026-10-23)
kbase_sources: [dataminer-query-viewer, dataminer-_index]
last_reviewed: 2026-10-23
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
| `TIMEOUT` "The query exceeded the server-side statement timeout." | The request ran past the server's limit. | The procedure below. Where the error carries a hint, it names what this tool can narrow. |
| `TIMEOUT` "The query waited too long for a table lock." | The nightly data refresh (the ETL run from 18:30 server time, [[concepts/data-freshness]]) holds the data. | Wait until the refresh has finished and retry the same request once. |
| The client's own request timeout (in Cowork: "Request timed out"), with no result from the MCP | The call outlived the client's ceiling. | Treat it like a statement timeout. A longer server limit does not help this client. |
| `INTERNAL` "timeout exceeded when trying to connect" | The server is busy: no database connection was free. | Retry once after a short pause; if it happens again, narrow the request. |

A `search_code` refusal for a pattern that matches too many code blocks is not a timeout: the search stops once it has counted too many matching blocks and lists none. Narrow as its hint says.

## The limits

- Every tool's queries on the data have a server-side statement limit of about half a minute. Where the MCP build has `search_code`, `execute_sql` and `search_code` get a longer one, about 50 seconds by default.
- The chat client adds its own ceiling. Cowork stops a tool call after 60 seconds.
- The appliance operator sets the server limits; the MCP server's admin documentation describes them.

## Procedure

1. **Narrow the request or change its shape.** One application instead of the estate, a design-document or code-element key instead of a text search, a more specific pattern, a smaller `top_n` or `limit`. For code text use `search_code` where the build has it; in SQL read `cqx_data.code_source` first and the locations second (two calls, see [[howtos/navigate-code]]). Most timeouts end here.
2. **Use the semantic tool that answers the question,** if there is one. Never rebuild a semantic tool's logic in SQL; for lifecycle decisions there is no SQL route at all.
3. **Still too long, and the question needs SQL:** the user can run the query in DataMiner, which is not bound by the MCP's limit. DataMiner opens from the Portal footer ("Open DataMiner") and depends on the user's DataMiner access. Give the SQL with a one-line description of what it shows; the user runs it there with ▶▶ (run all) for the full result - ▶ shows a preview with a row limit - and copies the result back into the conversation if needed. The Companion cannot read DataMiner results itself.
   - The SQL follows the session's rules like any SQL shown here: in a restricted session no per-user method - state what is not available and offer the aggregate alternative first; a question about a named individual stays a clean refusal, with no SQL and no alternative.
   - For a long query the user needs again: a saved DataMiner query can be scheduled on its Options tab for automatic nightly data preparation, so its result is ready without waiting (saving a query needs DataMiner's Editor Mode).
4. **For consultants working with the appliance operator:** on builds with `search_code`, the operator can raise the longer limit of `execute_sql` and `search_code` (`MCP_EXTENDED_STATEMENT_TIMEOUT_MS`; see the MCP server's admin documentation). It helps only with a client whose own tool-call timeout is longer: the GitHub Copilot app (a timeout per MCP server, 180 seconds by default) or Claude Code (`MCP_TOOL_TIMEOUT`, or the `timeout` of the server's entry, per its documentation). Other clients have their own setting. Cowork stops at 60 seconds whatever the server allows.

## Interpretation

- A statement or client timeout is not "the MCP is unavailable": do not fall back to the wiki's SQL patterns as if the server were down, and do not retry the same request unchanged (the lock-wait and connection rows above are the exceptions).
- During the appliance's nightly data refresh, queries run slower and lock waits happen: wait and retry once after it.
- Tell the user in plain words what happened (the request ran past the time limit) and what you tried, relay the hint where the error carries one, and offer the next step as an `AskUserQuestion`: narrow the request, use the semantic tool, or run the SQL in DataMiner.

## Variations

- "Can the Companion read my DataMiner results?" No: the MCP reads `cqx_data` only. Paste the result, or read it in DataMiner or Metabase.
- A semantic tool times out on one application: narrow its options (a smaller `top_n`, one ruleset or window); if that does not help, say so - the limit is set on the server, not per call.
