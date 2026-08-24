---
id: concepts/usage-time-windows
title: Usage over time windows — canonical buckets vs custom windows
type: concept
status: verified
tags: [domain/idna, topic/usage, audience/consultant]
related:
  - concepts/usage-tier
  - concepts/user-access-days
  - concepts/regulatory-usage-constraints
  - schema/tables/usage-cube-repset-activity-history
  - schema/tables/usage-cube-weekly-summary
  - schema/tables/replica-set
  - howtos/analyze-usage-over-custom-window
requires_capability: none
source: user-activity-time-windows plan (2026-07-27), external audit E3-E5; window-coverage fix (2026-08-24)
last_reviewed: 2026-08-24
---

# Usage over time windows

## What it is

iDNA pre-aggregates usage into **canonical horizons** — last 7 / 30 / 90 / 365
days and all recorded history — and builds its tier labels, lifecycle funnels,
and dashboards on them (see [[concepts/usage-tier]]). Users unfamiliar with this
routinely ask for windows they made up: "the last six months", "between March
and June", "this year". This page defines how the Companion answers those —
a **three-rung ladder**, with binding rules for when each rung applies.

## The three-rung ladder

1. **Canonical buckets** — pre-computed columns on
   [[schema/tables/replica-set]] (tiers, UAD windows, `users_active_last90d`).
   Instant, and — decisively — **comparable**: every tier label, keep/sunset
   threshold, and dashboard figure speaks this scale. Default for *vague*
   recency or comparability questions ("is this app still used much?").
2. **Custom window, weekly grain** — the `usage_over_window` MCP tool (series /
   rank / cross modes), backed by the weekly activity-history tables. Works in
   **every** access profile. Manual SQL patterns:
   [[howtos/analyze-usage-over-custom-window]].
3. **Per-user depth, daily grain** — the `user_activity_over_window` MCP tool
   (privileged profile only). Offered only after the active access profile is
   resolved. Recipes live on a permissive-only howto page and are not part of
   this page.

**The two window tools do not cover the same period.** `usage_over_window`
works on whole Monday-anchored weeks, so a custom window is widened to every
week it overlaps (it reports the real span as `covered_from_date` /
`covered_to_date`). `user_activity_over_window` works on daily session rows, so
its window is exact. Asking both for "July 2026" therefore covers
2026-06-29–2026-08-02 in the first and 2026-07-01–2026-07-31 in the second:
their numbers are not directly reconcilable. Use the weekly tool for aggregate
trend and restricted-safe work, the daily tool when the window boundaries must
be exact — and say which one a figure came from.

## The explicit-window rule

When the question names a **concrete non-canonical window** — "last six
months", "between March and June", "since 2026-03-01" — that *is* the request
for rung 2. Answer the custom window directly and present the nearest canonical
bucket **alongside** it. Do not substitute a bucket-only answer: for seasonal
applications, migration cutover windows, or incident windows, a 90d or 365d
figure is a materially *wrong* answer, not a slower one.

Bucket-first redirection remains correct for vague phrasing ("recently",
"these days") — explain the horizons, offer the custom window as an option.

**Anti-escalation:** questions rung 1 already answers fully must not invoke
window machinery. "Has anyone used app X since March?" is a recency question —
`last_access` / `days_since_last_access` answers it in one lookup.

## The comparability guard

Custom-window totals and rankings are **bespoke context only**:

- You may *compare* a custom-window figure with the nearest canonical bucket
  (the tools embed that context for exactly this purpose).
- You must **never assign** a usage tier or percentile label from a
  custom-window total. Tiers are calibrated per environment on canonical
  horizons; a six-month UAD mapped onto them is a category error. A seasonal
  app showing `usage_last90d = 'Low'` during its quiet months while its
  enrollment-month window shows heavy use is *expected shape*, not a
  contradiction — report both, grade neither.
- Custom windows **enrich** the canonical picture; they never replace it.

## The headcount asymmetry

Additive metrics — `user_access_days`, sessions, reads, writes, `days_active` —
sum exactly across weeks. **`users_active` does not**: it is a per-week
distinct count, and the same user recurs across weeks. Over a custom window,
aggregate tables can report the **peak week** only (a floor, not the
window-distinct headcount). State the limitation first, give the peak-week
figure and the aggregate answer; exact window-distinct user counts require
per-user data — if the customer has already approved the privileged MCP
posture, `user_activity_over_window` computes them.

Weekly grain itself has a floor: iDNA stores usage in whole Monday-anchored ISO
weeks. A custom window is therefore widened to every week it overlaps — no
requested day is lost, but the first and last weeks can reach outside the
window. The tool reports the real span as `covered_from_date` /
`covered_to_date` and says so in its notes whenever the widening is non-empty;
quote that span when you report the numbers. A Monday–Sunday window is not
widened.

## Profile gating and the no-fallback rule

Rung 3 is privileged-only, at three enforcement layers (runtime tool gate,
manifest, Postgres role — see [[concepts/regulatory-usage-constraints]]).
Under the restricted profile:

- App-, department-, and location-pivoted questions degrade to a real
  aggregate answer: same question, coarser grain (rung 2 cross mode — "which
  departments", not "which users").
- **User-pivoted questions ("what did person N do?") have NO restricted-mode
  fallback.** The correct response is a clean refusal with the reason. A
  department- or location-pivoted question is a *different question* the user
  may choose to ask — its answer must not be presented as an answer about the
  individual. Cohort-as-proxy inference is circumvention of the customer's
  compliance posture, not helpful degradation.

Wording rule for every restricted-mode mention of the privileged path: state
the limitation first, give the aggregate alternative, and mention the
privileged capability only as *"if the customer has already approved that
posture"*. Never prescribe enabling it.

## Related

- [[concepts/usage-tier]] — the canonical horizons and tier labels.
- [[concepts/user-access-days]] — the metric being summed.
- [[howtos/analyze-usage-over-custom-window]] — rung-2 recipes.
- [[schema/tables/usage-cube-repset-activity-history]] /
  [[schema/tables/usage-cube-weekly-summary]] — the weekly backing tables.

## Sources

Design and audit trail: `docs/plans/active/user-activity-time-windows/`
(dev-repo); explicit-window rule and comparability guard locked by external
audit (E3/E4), wording posture by E5. Data facts verified against the iDNA ETL
2026-07-27.
