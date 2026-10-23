---
id: concepts/data-freshness
title: Data freshness — the nightly ETL and what "as of" means
type: concept
status: verified
tags: [domain/idna, topic/usage, audience/consultant]
related:
  - concepts/usage-time-windows
  - concepts/usage-tier
  - concepts/lifecycle-vector
  - schema/tables/replica-set
  - playbooks/application-lifecycle/lifecycle-routing
  - howtos/handle-query-timeouts
requires_capability: none
source: owner statement on the ETL schedule (2026-10-23: fixed at 18:30 server-local time, customers cannot move it); ETL etl-manifest.mfx (refresh timeframe 0 30 18 * * ?); usage_data_freshness semantics from the MCP server (introspection.ts); trial last_accessed per ETL dda_dm/scheduled/230
last_reviewed: 2026-10-23
---

# Data freshness

## What it is

Everything the Companion reads — `cqx_data` tables, the lifecycle and
modernization vectors, the dashboards — is produced by a **nightly ETL run**.
The run starts at **18:30 in the iDNA server's local time** on every iDNA
installation - the shipped ETL schedule, which customers cannot move - and
processes the **previous full calendar day** of collected activity. It does not process the current day.

Consequence: the newest activity date in the data is normally one or two days
behind today.

| Time of day (iDNA server local time) | Last completed run | Newest activity day in the data |
|---|---|---|
| before today's run has completed | yesterday 18:30 | the day before yesterday (today − 2) |
| after today's run has completed (it starts at 18:30) | today 18:30 | yesterday (today − 1) |

Example: on 23 October before 18:30, the last run was 22 October 18:30, which
processed 21 October. The data is "as of 21 October" and that is expected,
not a fault.

The Companion sees neither the server's clock nor the run itself, so take the
row from the field: today − 1 means today's run has completed, today − 2 that it
has not yet, or that yesterday saw no activity. Near midnight, or when the user and the iDNA server are in
different time zones, "today" can differ by one day; say so rather than calling
the gap abnormal.

## How to read `usage_data_freshness`

`get_idna_session` returns `usage_data_freshness`, the most recent
`last_accessed` date across all replica sets
([[schema/tables/replica-set]]). It measures **observed activity**, not the
ETL run itself. Three things can make it older than today − 2:

- **The ETL has not run.** Every day without a run adds a day to the gap.
- **No activity was observed.** On a quiet estate (weekends, holidays), or
  when collection from the Domino servers has stalled, the ETL runs but finds
  nothing newer.
- **On a trial licence the field covers the trial-selected applications only**
  (`last_accessed` is empty for the others), so a quiet sample makes the gap
  larger.

The field cannot tell these apart. `null` means no `last_accessed` date at
all: no usage data has landed yet (typically a freshly onboarded tenant) or, on
a trial licence, none of the selected applications has recorded use.

## Rules for the Companion

- **Explain, don't warn, for a gap of 1–2 days.** When a user asks why the
  data is not from today, state the cadence: the nightly 18:30 run covers the
  previous full day. Give the concrete dates for the session.
- **Treat a gap of more than 2 days as a question, not a conclusion** - raise
  it once per session, when usage figures are reported or the user asks. Name
  the gap and the possible causes above; point to iDNA's collection and
  ETL status rather than guessing which one applies. The lifecycle preflight
  in [[playbooks/application-lifecycle/lifecycle-routing]] escalates to a
  warning at more than 7 days.
- **`null`:** say that no usage date is available for this session and give
  the possible reasons above; lifecycle work treats it like the 7-day warning
  ([[playbooks/application-lifecycle/lifecycle-routing]]).
- **Never compensate.** Do not shift windows, extrapolate "today" from older
  days, or describe today − 1 / today − 2 data as stale. Canonical windows
  ([[concepts/usage-time-windows]]) are computed by the ETL from the run's
  date, so they end with the processed day and are internally consistent.
- **"As of" wording.** When reporting usage figures, "as of
  <usage_data_freshness>" is the correct qualifier. Do not say "as of today".
