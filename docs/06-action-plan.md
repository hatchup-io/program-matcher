# ⑥ Action plan

> Sibling docs: [collection](01-data-collection.md) · [storage](02-data-storage.md) · [update](03-data-update.md) · [questions](04-client-questions.md) · [delivery](05-provider-delivery.md)

Six workstreams. Sizes are engineering weeks for one backend engineer, excluding review latency. Every task is scoped to land as one commit, matching the platform's convention.

## Overview

| # | Workstream | Depends on | Size | Ships |
|---|---|---|---|---|
| **W1** | Storage tiers | — | 1.5 wk | Migrations, models, reindex command |
| **W2** | Collection and review | W1 | 2.5 wk | A program authored and approved end to end |
| **W3** | Update and integrity | W1 | 1 wk | Index provably matches the catalog |
| **W4** | Question loop | W1 | 2 wk | Behind a flag, shadow first |
| **W5** | Provider delivery | — | 1.5 wk | Catalog inverted to program-first |
| **W6** | Evaluation harness | W4 | 1 wk | The numbers that decide whether W4 ships |

**W5 depends on nothing here and should start immediately.** It is the change that makes the system program-first, it is the cheapest item on the list, and it is independently valuable even if everything else slipped.

## W1 — Storage tiers · 1.5 wk

1. Enable the `pgvector` extension; add the migration and the deployment note.
2. `ProgramSource` model and migration.
3. `ProgramRevision` model, with `facts` and `approved_fields`, plus the `current_revision` pointer on the program.
4. `ProgramChunk` model with the HNSW index and the section-type taxonomy.
5. Section chunker over the existing document set — pure function, unit-tested against three real documents.
6. Embedding client with batching, retry and a model name read from settings rather than hard-coded.
7. `manage.py reindex_programs [--program] [--force]`.
8. Backfill: the seventeen seeded programs published as revisions against a `legacy_seed` source, flagged for review.

**Acceptance.** A full rebuild from Tier 3 reproduces identical content hashes. The eligibility path contains no Tier-2 read. No behavioural change to any existing verdict.

## W2 — Collection and review · 2.5 wk

1. Source registration API with mandatory provenance.
2. Extraction agent producing per-field proposals with source spans and confidence.
3. Per-category required-field contract, expressed as data and enforced on publish.
4. Field review API — approve, edit with reason, mark unknown.
5. Review panel UI: document beside proposals, one keystroke per field.
6. Coverage report endpoint and panel view.
7. Provider self-authoring path, reusing the same gates with provider scoping.
8. Conflict surfacing when two sources disagree on a legal fact.

**Acceptance.** A real program goes from PDF to published without a developer touching a file. Publish is blocked, with the field named, while any required field is unapproved. Approval-edit rate is measurable from day one.

**Gate — end of week 3.** Does the intake gate hold when a real program goes through it? If a reviewer edits most proposed fields, extraction is not helping and W2 narrows to provenance plus manual authoring.

## W3 — Update and integrity · 1 wk

1. Publish as a single transaction, with supersede and hash-based chunk reuse.
2. Rollback, and the interruption path for sessions pinned to a rolled-back revision.
3. Session revision pinning, read through retrieval, eligibility and presentation.
4. Revision id recorded on saved decision cards and on booking attribution.
5. Nightly drift scan with its seven assertions, reporting without repairing.
6. `review_due` per source kind, with the panel review queue and the *verified as of* line on the card.

**Acceptance.** A one-field change re-embeds exactly one chunk. A session pinned to a revision produces identical output before and after a later publish. Each of the seven drift failures is detected and reported in a test.

## W4 — Question loop · 2 wk

1. `candidate_state` on the session, with exclusion reason codes and turn numbers.
2. Signal catalog as data: type, values, source, cost, language labels.
3. Question bank with `en` and `fa` labels, completeness enforced by test.
4. Information-gain selection, with the zero-gain skip.
5. Stop conditions, with the existing phase plan retained as cold-start ordering and fallback.
6. Retrieval integration: pre-filtered query, per-program cap.
7. Grounding guardrail: reject a program claim with no supporting chunk.
8. Shadow mode: run the loop alongside the live chat, logging what it would have produced.
9. Scenario handling: hard block early, contradiction reopen, refusal terminal, applicant question answered from chunks.

**Acceptance.** Every criterion in [questions §4.10](04-client-questions.md). Nothing user-facing until W6 reports.

## W5 — Provider delivery · 1.5 wk

1. Invert the catalog service: program-first browse, category-scoped, not provider-scoped.
2. `delivery_options()` resolution with the exclusivity shortcut and the funnel filter.
3. Delivery endpoints, including the ordering inputs on the response.
4. Provider card assembly, with response time derived from booking history.
5. Not-deliverable state and the unmet-demand counter.
6. Booking attribution derived from the matched program and chosen assignment, including the revision id.
7. Ordering rule — **blocked on the business decision**, defaulting to rating with capacity tie-break until it lands.

**Acceptance.** Every criterion in [delivery §5.9](05-provider-delivery.md), including that an existing provider-only funnel behaves identically against the current test suite.

## W6 — Evaluation · 1 wk

1. Golden set: ~50 applicant profiles labelled by a consultant with the program they should reach.
2. Replay harness running a profile through the loop without a live model where possible.
3. Metrics: turns to shortlist, top-1 and top-3 agreement, non-discriminative turns, unsourced claims, drift.
4. Shadow comparison report: loop versus current ranker on real sessions.

**Gate — end of week 6.** Does the loop beat the current ranker on the golden set? Nothing user-facing ships before this reports.

| Metric | Baseline | Target |
|---|---|---|
| Turns to shortlist | up to 36 | ≤ 10 median |
| Top-1 agreement with consultant label | measure in shadow | ≥ 0.8 |
| Top-3 agreement | measure in shadow | ≥ 0.95 |
| Non-discriminative turns per session | measure in shadow | ≤ 1 |
| Program claims with no source | unmeasured | 0 |
| Index-vs-catalog drift | unmeasured | 0 |

## Sequence

```
week   1         2         3         4         5         6
       ├─ W1 ─────┤
       │          ├─ W2 ───────────────────────┤
       │          ├─ W3 ──┤
       │          ├─ W4 ──────────────┤
       │                              ├─ W6 ───┤
       ├─ W5 ──────────┤
                       ▲                       ▲
                   gate 1                   gate 2
              intake holds?            loop beats ranker?
```

Two decision points, not one deploy.

## What we need from the business

| # | Decision | Blocks | Recommendation |
|---|---|---|---|
| 1 | Provider ordering rule for non-exclusive programs | W5 final slice | Rating first, capacity tie-break; commercial terms out of ordering |
| 2 | Who approves a field at intake | W2 | Provider authors, internal approves anything legal |
| 3 | Consultant time for the golden set (~50 profiles) | W6 | Two half-days |
| 4 | Review cadence per source kind | W3 | 12 months official, 6 expert, 3 provider terms |

Items 1, 2 and 4 have working defaults, so nothing stalls waiting for them. **Item 3 has no default** — without labelled profiles there is no way to say whether any of this worked.

## Risks

| Risk | Mitigation | Detected by |
|---|---|---|
| Extraction quality too low to be worth reviewing | Per-field approval; W2 narrows to manual authoring | Approval-edit rate, gate 1 |
| Retrieval biased toward verbose documents | Per-program cap in the query | Shadow comparison |
| Over-pruning a program a consultant would have kept | Reason code on every exclusion; near-miss tier | Golden set top-3 |
| Loop converges fast but wrongly | Nothing ships before gate 2 | Top-1 agreement |
| Index drifts from catalog | Same-transaction writes; nightly scan | Drift metric |
| Scope creep into the locked flow | Locked flow takes grounding only, not the loop | Review |
