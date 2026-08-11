---
id: concepts/design-families
title: Design Families
type: concept
status: verified
tags: [domain/idna, topic/modernization]
related:
  - concepts/design-similarity
  - concepts/about-modernization-vector
  - concepts/about-modernization-plan
  - concepts/about-template-inheritance
  - schema/tables/modernization-vector
requires_capability: none
source: ETL dda_dm/scheduled/284_dda_dm.modernization_design_families.update.sql + ADR 0001 (iDNA_Applications_ETL docs/decisions)
last_reviewed: 2026-07-23
---

# Design Families

## What they are

A **design family** is a transitive connected component over iDNA's design-similarity clusters (see [[concepts/design-similarity]]): if A is ≥85% design-similar to B and B to C, then {A, B, C} form one family — even if A and C were never directly compared. Families are computed for **all focus replica sets** (any lifecycle) and surface on `cqx_data.modernization_vector` as `design_family_id`, the `design_family` replicaid array, and the peer counts (`peers_keep_modernize`, `peers_keep_qualify`, `peers_sunset`).

## Why they exist (the missed multiplier)

Design-similar apps *without* template links are structurally siblings that nobody wired together — ten copies of the same app, each maintained separately. That is a **missed multiplier**: instead of fixing 10 databases one by one, unify them under one new shared master template — the clean way to manage applications in Notes/Domino. The [[concepts/about-modernization-vector]] routes such apps to `template/unify_similar` (requires ≥1 kept peer), and the plan turns each family into a unification session.

The decision itself is a human one: locate where the family members diverge (block-level, via the worklist), judge whether the differences are reconcilable, and accept that **partial outcomes are legitimate** — a 10-member family may end up under two masters, or prove not reconcilable at all. iDNA supplies the family roster and the block evidence; the unification is developer work outside iDNA.

## Similarity is not hash identity (the drift trap)

Family membership is **design similarity** at ≥85% cluster grain — it tolerates code drift. Code-block hashes do not: drifted variants of the same logical script have *different* hashes, so "design similar" does **not** imply "one patch applies across the family". A family member can show shared/solo counts of 8/0 while its sibling shows 4/4 — the sibling carries block *versions* nobody else has (drift → different hash → solo). When family numbers "feel weird", reconcile per block hash; the columns measure estate-wide patch potential, not family commonality.

## Sunset peers

Families include sunset members deliberately: they appear in the `design_family` array, the `peers_sunset` count, and session displays ("+N sunset peer(s)"). They get no vector rows and no sessions of their own ([[concepts/lifecycle-vector]] decides keep vs. sunset), but they are *evidence* — their existence informs the unification decision, and their findings count in the **vector's estate-wide shared/solo reach** (NOT in the worklist's `reach`, which counts kept apps only — the two scopes are deliberately different).
