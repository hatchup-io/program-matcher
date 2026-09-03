---
marp: true
theme: default
paginate: true
size: 16:9
title: Program Matcher — a goal-directed chat for Bayat Group
description: Design proposal — turn the guided chat from open discovery into a targeting loop over a vector-indexed program catalog
style: |
  section {
    font-size: 25px;
    padding: 48px 60px;
  }
  section.lead {
    justify-content: center;
    text-align: left;
  }
  h1 { color: #12324f; font-size: 46px; }
  h2 { color: #12324f; font-size: 34px; border-bottom: 2px solid #e2e8f0; padding-bottom: 6px; }
  code { font-size: 0.82em; }
  pre { font-size: 0.68em; line-height: 1.35; background: #f7fafc; border: 1px solid #e2e8f0; }
  table { font-size: 0.78em; }
  strong { color: #0f766e; }
  blockquote { border-left: 4px solid #0f766e; color: #334155; font-style: normal; }
  footer { color: #94a3b8; font-size: 15px; }
footer: 'Bayat Group — Program Matcher — internal engineering'
---

<!-- _class: lead -->
<!-- _paginate: false -->

# Program Matcher

## A goal-directed chat for Bayat Group

Turning open-ended discovery into a **targeting loop** over a vector-indexed program catalog.

Design proposal — internal engineering — September 2026

---

## The ask, in one slide

The chat should stop *exploring* and start *converging*.

- **Explicit target.** Every session has one job: identify the program (or the two or three) that fit this applicant, and say why.
- **Every turn earns its place.** A question is asked only when its answer can still change the outcome.
- **Programs live in a vector store.** The catalog and its knowledge base become retrievable, so the conversation is anchored to real programs from the second turn, not only at the end.
- **The engines stay deterministic.** Eligibility, qualification and ranking remain code. Retrieval narrows the field; it never decides.

---

## Where we are today

`apps/funnel_modules/bayat_group/` — the guided flow, as built:

| Piece | File | What it does |
|---|---|---|
| Phase plan | `general_chat_phases.py` | 6 fixed phases, up to 6 turns each |
| Engine | `phased_chat.py` | `ExploratoryCompletion` — advance on signals **or** turn budget |
| Eligibility | `programs/eligibility/{cbi,rbi,immigration}.py` | Data-driven hard gate per category |
| Ranking | `ranking.py` | Priorities × per-program objective ratings |
| Presenting | `presenter.py` | `present_shortlist()` — the ranked result |

The catalog is **17 seeded programs** across CBI / RBI / Immigration, plus a rich markdown knowledge base in `docs/bayatgroup-prgorams/` that **nothing in the runtime currently reads**.

---

## The problem, precisely

Three distinct failure modes, all from the same root: **the conversation has no notion of the answer it is looking for.**

1. **Fixed-length discovery.** Six phases run to their budget whether or not the outcome is already determined. Up to ~36 turns to reach a shortlist that was decidable at turn 8.
2. **Non-discriminative questions.** The plan asks what a *consultation* asks, not what *separates the surviving candidates*. If every remaining candidate is CBI, the residency-versus-passport question costs a turn and changes nothing.
3. **Late, unanchored recommendation.** Ranking happens after the chat ends. Nothing in the transcript is grounded in real program facts, so the agent talks around programs it is not allowed to name yet.

The expert knowledge base is offline. The chat cannot cite what Bayat Group actually knows.

---

## The target: an explicit objective function

Give the session a state and a goal, and the conversation follows from them.

```
state      C  = candidate programs still viable
           B  = belief over the applicant (signals captured so far)
goal       reduce |C| to a decided target set, honestly
per turn   ask the question that most reduces H(C)
stop       when no remaining unknown can reorder C
```

This is the whole change. Everything else — vectors, retrieval, grounding — serves this loop.

> The chat stops when the answer is found, not when the script runs out.

---

## Architecture at a glance

```
                     ┌───────────────────────────────────────────┐
  program seeds  ──▶ │  INGESTION                                │
  docs/bayatgroup-   │  chunk → embed → upsert (content_hash)    │
  prgorams/**.md     └──────────────────┬────────────────────────┘
                                        ▼
                          ┌─────────────────────────────┐
                          │ pgvector: program_chunks    │
                          │ embedding + program_id +    │
                          │ category + section + text   │
                          └──────────────┬──────────────┘
                                         ▼
  applicant turn ─▶ ┌──────────────────────────────────────────┐
                    │ TARGETING LOOP                           │
                    │ 1. filter  (hard constraints, SQL)       │
                    │ 2. retrieve(dense + metadata → C)        │
                    │ 3. score   (eligibility + qualification) │──▶ shortlist
                    │ 4. select  (max information gain → ask)  │    + citations
                    │ 5. stop?   (stability / dominance)       │
                    └──────────────────────────────────────────┘
```

---

## Layer 1 — the catalog becomes retrievable

**Sources.** Per-program docs in `docs/bayatgroup-prgorams/{CBI,RBI,Immigration}/**`, the `facts` / `requirements` / `objectives` blocks on `MigrationProgram`, and the Sam Bayat methodology briefing.

**Chunking.** By document section, not by fixed window — these docs are already sectioned (routes, physical presence, family inclusion, restricted nationalities, processing). A section is the unit an advisor cites.

**Metadata on every chunk.** `program_id`, `category`, `country_code`, `section_type`, `source_path`, `source_revision`. The metadata is what makes retrieval *filterable*, and filtering is what keeps it honest.

**Freshness.** `content_hash` per chunk; a `post_save` on `MigrationProgram` and a `just reindex` management command. A seed edit that does not reach the index is a silent wrong answer.

---

## Why pgvector, not a dedicated vector DB

We already run PostgreSQL 18 on the shared cluster. `pgvector` with an HNSW index gives us:

- **One store, one transaction.** A program row and its chunks commit together. No dual-write, no reconciliation job, no "the index says Portugal, the catalog says it was retired".
- **Filters and vectors in one query.** Hard constraints (nationality bans, contribution floor) are SQL predicates on the same table — pre-filtering, not post-filtering a wrong top-k.
- **Nothing new to operate.** No extra service in the swarm, no extra backup rotation, no extra failure mode on a deploy.

At 17 programs and a few thousand chunks, a dedicated vector database buys latency we do not need and costs operational surface we would feel immediately. Revisit at ~10⁵ chunks or when a second consumer appears.

---

## Honest framing: what the vectors actually buy

Worth saying plainly to avoid building the wrong thing.

- **Vector search is not what makes the chat converge.** With 17 programs, the ranker could enumerate the catalog. The **information-gain loop** is what ends the wandering.
- **What retrieval buys is grounding and reach.** It puts the expert knowledge base into the conversation — cited, current, and per-program — and it keeps working when the catalog is 200 programs and provider self-service (Phase 7) is authoring them.
- **Build both, but know which is which.** If we shipped only the vector store, the chat would still wander. If we shipped only the loop, the chat would converge but still could not cite a source.

---

## Layer 2 — the targeting loop

Per turn, in order:

1. **Filter.** Apply captured hard constraints as SQL predicates — sanctioned nationality, contribution floor, dependent age limits. Programs excluded here are excluded with a reason code, never silently.
2. **Retrieve.** Embed the accumulated intent narrative; dense search over surviving programs' chunks; aggregate chunk scores to a per-program score with a per-program cap so a verbose document cannot outvote a terse one.
3. **Score.** Feed candidates through the existing `resolve_eligibility()` and the qualification scorer. Unchanged code, unchanged verdicts.
4. **Select.** Choose the next question by expected information gain over the candidate set.
5. **Stop.** Test the stop conditions before generating another question.

---

## Question selection by information gain

For each unanswered signal `s`, partition the candidate set by the answers `s` can take, and score the split.

```
gain(s) = H(C) − Σ  P(v) · H(C | s = v)
                v∈values(s)

ask  argmax gain(s) · confidence(s) / cost(s)
```

- `cost(s)` — a question about a passport is cheap; one about undisclosed funds is expensive and is asked late, or only when it is decisive.
- **Zero gain ⇒ never asked.** If all surviving candidates are CBI, `primary_goal ∈ {residency, second_passport}` no longer splits anything, and the loop skips it.
- The existing phase plan stays as the **fallback ordering** for the cold start and for when retrieval returns nothing.

---

## A worked example

Applicant: Iranian national, EUR 300k liquid, wants the family in the EU within a year.

| Turn | Signal captured | Candidates | Why this question |
|---|---|---|---|
| 1 | `primary_objective = residency` | 17 → 8 | Highest gain at cold start — splits CBI from RBI/Immigration |
| 2 | `nationality = IR` | 8 → 5 | Hard filter; three programs excluded with a stated reason |
| 3 | `contribution_range = 250–500k` | 5 → 3 | Contribution floor is the sharpest remaining split |
| 4 | `family = spouse + 2 children` | 3 → 3 | No reordering, but sets the plan tier for the verdict |
| 5 | `timeline = within 12 months` | 3 → 2 | Processing time separates the leaders |

**Stop at turn 5**, not turn 36. The two survivors are presented with citations, and the rest of the discovery moves into the roadmap form where it belongs.

---

## Stop conditions

The loop ends when any of these holds — and it must be able to end:

- **Dominance.** The top candidate leads by a margin no remaining unknown can close.
- **Stability.** The top-k has not changed for *n* turns and the open signals are non-discriminative.
- **Exhaustion.** No unanswered signal has positive gain.
- **Empty.** No program survives the hard gates — present that honestly and route to a consultant, which is the existing `NOT_ELIGIBLE` path.
- **Budget.** A hard ceiling stays, as a backstop rather than as the mechanism.

Reaching a stop condition produces the shortlist card. Everything the verdict still needs but the chat did not ask is collected by the roadmap form.

---

## What the LLM does — and does not do

Unchanged from the current locked decision, and worth restating because retrieval invites the opposite.

**The model does:** run the conversation, extract signals from natural language, phrase the selected question, and present the result in the category's voice.

**The model does not:** choose the candidate set, score eligibility, compute qualification, order the shortlist, or state a program fact that is not in a retrieved chunk.

> Retrieval narrows. Deterministic engines decide. The model only speaks.

---

## Grounding and citations

Every program claim the agent makes carries the chunk it came from.

- The turn prompt receives the retrieved chunks for the surviving candidates and **nothing else** about programs.
- A claim with no supporting chunk is a prompt-level violation; the guardrail pass in `guardrails.py` is extended to catch it, the same way it already screens input.
- The shortlist card gains a `sources` block — document path plus section — so a consultant reading the transcript can verify a number against Bayat Group's own file.

This is not polish. It is a regulated advisory product quoting investment thresholds and processing times.

---

## Data model changes

```python
class ProgramChunk(TimeStampedModel):
    program      = FK(MigrationProgram, related_name="chunks")
    section_type = CharField(...)        # routes | presence | family | ...
    content      = TextField()
    content_hash = CharField(db_index=True)
    source_path  = CharField()           # docs/bayatgroup-prgorams/...
    embedding    = VectorField(dims=3072)  # text-embedding-3-large
    # HNSW index on embedding; btree on (program_id, section_type)
```

On the session, alongside the existing `signal_state`:

- `candidate_state` — surviving slugs, per-program score, exclusion reasons, and the turn each exclusion happened on. It is the audit trail for *why this shortlist*, and it is what makes the loop debuggable in the operator panel.

---

## Rollout — four phases, each shippable

| Phase | Ships | Risk |
|---|---|---|
| **0 — Index** | `pgvector`, `ProgramChunk`, ingestion command, reindex on save. No runtime consumer. | None — additive |
| **1 — Shadow** | Loop runs alongside the current chat, logging the shortlist it *would* produce. Nothing user-visible. | None — measurable |
| **2 — Guided flow** | Targeting loop drives the guided chat behind a funnel flag. Phase plan is the fallback. | Contained, reversible |
| **3 — Grounding** | Retrieved chunks + citations in the locked chat and the shortlist card. | Prompt-level |

Phase 1 is the one that decides whether this is worth finishing. We compare against the current ranker on real sessions before a single user sees a different question.

---

## Evaluation — or it is just an opinion

A golden set of ~50 applicant profiles labelled by a consultant with the program they should reach.

| Metric | Today (baseline) | Target |
|---|---|---|
| Turns to shortlist | up to 36 | ≤ 10 median |
| Top-1 agreement with consultant label | measure in Phase 1 | ≥ 0.8 |
| Top-3 agreement | measure in Phase 1 | ≥ 0.95 |
| Non-discriminative turns per session | measure in Phase 1 | ≤ 1 |
| Unsourced program claims | unmeasured | 0 |

The harness lands in Phase 1, before the loop is user-facing. "The chat feels more directed" is not a result.

---

## Risks and open questions

- **Index drift.** A seed edit that misses the index produces a confident wrong answer. Mitigated by `content_hash` plus reindex-on-save; needs a periodic consistency check as a Beat task.
- **Retrieval bias.** Verbose documents attract more chunk hits. Mitigated by the per-program cap; verify in shadow mode.
- **Over-pruning.** Aggressive filtering can drop a program a consultant would have kept. Every exclusion carries a reason code, and the shortlist keeps a "near miss" tier.
- **Cost.** One-off embedding of the catalog is negligible; per-turn query embeddings are cached by intent hash.
- **Open:** does the locked (self-select) flow adopt the loop, or only grounding? Proposal: grounding only — its target is already chosen.
- **Open:** does the shortlist ever return a single program, or always show the runners-up? Proposal: always show, with the margin stated.

---

## What we are asking for

1. **Agreement on the framing** — the chat's job is to find a program, and turns that cannot change the answer are not asked.
2. **Green light for Phase 0 + 1** — the index and the shadow comparison. Additive, invisible to users, and it produces the numbers the rest of the decision needs.
3. **A consultant hour** for the golden set. Fifty labelled profiles is the difference between a measured improvement and a hunch.

Spec and open threads: `docs/ARCHITECTURE.md` in this repository.
