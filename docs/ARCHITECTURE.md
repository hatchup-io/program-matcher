# Program Matcher — architecture

> Status: proposal. Nothing here is built. The deck in `deck/program-matcher.md` is the summary; this file is the detail behind it.

The implementation target is `hatchup-io/bayatgroup-backend`, in `apps/funnel_modules/bayat_group/` and `apps/programs/`. This repository holds the design so it can be reviewed before code exists, not because the system will live here.

## 1. What we are changing

The Bayat Group guided chat runs a fixed six-phase discovery plan (`apps/funnel_modules/bayat_group/general_chat_phases.py`) on the shared phased-chat engine (`apps/funnel_modules/phased_chat.py`), with `ExploratoryCompletion(max_turns_per_phase=6)` as the completion policy. Programs are ranked only after the chat finishes, by `ranking.py`, against the applicant's stated optimization priorities.

That design gathers a consultation. It does not converge on a program, and it cannot: the conversation carries no representation of which programs are still viable, so no turn can be skipped on the grounds that it no longer changes the answer.

The change is to give the session an explicit candidate set, drive question selection from it, and back it with a retrievable index of the program knowledge base so the conversation can be grounded in real program facts from the second turn.

## 2. Two layers, in dependency order

### 2.1 Layer 1 — the retrievable catalog

The knowledge base already exists as markdown under `docs/bayatgroup-prgorams/{CBI,RBI,Immigration}/**` in the backend repo, alongside the structured `facts`, `requirements` and `objectives` blocks that the seeds in `apps/programs/seeds/*.yaml` load onto `MigrationProgram`. Nothing in the runtime reads the markdown today.

Ingestion chunks those documents **by section rather than by fixed window**, because the documents are already sectioned the way an advisor cites them — qualifying routes, physical presence, family inclusion, restricted nationalities, processing time, permanent residence and citizenship pathway. A fixed-window chunker would split a route's amount from its conditions, which is exactly the pair a wrong answer comes from.

Each chunk carries `program_id`, `category`, `country_code`, `section_type`, `source_path`, `source_revision` and a `content_hash`. The metadata is what makes retrieval filterable, and pre-filtering on hard constraints is what stops the store from returning a top-k that a nationality ban already excluded.

Storage is `pgvector` in the existing PostgreSQL 18 cluster, with an HNSW index on the embedding column and a btree on `(program_id, section_type)`. Embeddings come from OpenAI `text-embedding-3-large` (3072 dims), through the same key handling as the agents — with the model name read from `AppSettings`, matching the existing rule that the model is a database value and the env var is only a fallback.

Freshness is enforced two ways: a `post_save` hook on `MigrationProgram` marks its chunks stale, and a `manage.py reindex_programs` command rebuilds by `content_hash` so an unchanged chunk is never re-embedded. A periodic consistency check belongs in `apps/notifications/beat_schedules.py` alongside the other scans — a chunk set that silently diverges from the catalog is a confidently wrong answer, which is worse than an error.

### 2.2 Layer 2 — the targeting loop

State on the chat session, next to the existing `signal_state`:

```
candidate_state = {
  "surviving":  {slug: {"score": float, "verdict": str, "band": str}},
  "excluded":   {slug: {"code": str, "turn": int}},
  "asked":      [signal_key, ...],
  "updated_at": iso8601,
}
```

Per turn, in order:

1. **Filter.** Captured hard constraints become SQL predicates — sanctioned nationality, contribution floor, dependent age limits. Exclusions are recorded with a reason code and the turn number, never dropped silently.
2. **Retrieve.** The accumulated intent narrative is embedded and searched against the surviving programs' chunks. Per-program scores aggregate their chunk scores under a per-program cap, so a long document cannot outvote a terse one.
3. **Score.** Candidates go through the existing `resolve_eligibility()` and the qualification scorer unchanged. Their verdicts are the authority; retrieval only decides who gets scored.
4. **Select.** The next question is the unanswered signal with the highest expected information gain over the candidate set, discounted by how expensive that question is to ask.
5. **Stop.** The stop conditions are tested before another question is generated.

Question selection scores each unanswered signal `s` by the entropy reduction its answer would produce over the candidate set, weighted by extraction confidence and divided by a per-signal cost — a passport question is cheap, a source-of-funds question is expensive and is asked late or only when decisive. A signal whose answer cannot reorder the surviving set has zero gain and is never asked, which is the mechanism that removes the wandering.

The existing phase plan survives as the cold-start ordering and as the fallback when retrieval returns nothing. It stops being the thing that decides when the chat ends.

Stop conditions: **dominance** (the leader's margin exceeds what any remaining unknown can close), **stability** (top-k unchanged for *n* turns with only non-discriminative signals open), **exhaustion** (no positive-gain signal remains), **empty** (nothing survives the hard gates — the existing `NOT_ELIGIBLE` path, presented honestly and routed to a consultant), and a **hard turn ceiling** kept as a backstop rather than as the mechanism.

Whatever the verdict still needs but the loop never asked is collected by the roadmap's gathered-data form, which already prefills from `writeback_general_objectives()`.

## 3. What stays deterministic

Unchanged, and restated because retrieval invites the opposite: the model runs the conversation, extracts signals, phrases the selected question, and presents the result in the category voice from `voice.py`. It does not choose the candidate set, score eligibility, compute qualification, order the shortlist, or assert a program fact that is not in a retrieved chunk.

Grounding is enforced at the prompt boundary. The turn prompt receives the retrieved chunks for surviving candidates and no other program content, and the guardrail pass in `guardrails.py` is extended to flag a program claim with no supporting chunk, the same way it already screens input. The shortlist card from `presenter.present_shortlist()` gains a `sources` block carrying document path and section, so a consultant reading a transcript can verify a threshold against Bayat Group's own file.

## 4. Why pgvector rather than a dedicated vector database

One store means a program row and its chunks commit in the same transaction, so there is no dual-write and no reconciliation job. Filters and vectors live in one query, so hard constraints pre-filter instead of post-filtering a top-k that was already wrong. And nothing new joins the swarm — no extra service, no extra backup rotation, no extra way for a deploy to fail.

At seventeen programs and a few thousand chunks, a dedicated vector database buys latency we do not need at a cost in operational surface we would feel on the next deploy. The decision is worth revisiting at roughly 10⁵ chunks, or as soon as a second service needs the same index.

## 5. The honest framing

Vector search is not what makes the chat converge. With seventeen programs the ranker could enumerate the catalog on every turn; the information-gain loop is what ends the wandering, and it would work over a plain queryset.

What retrieval buys is grounding and reach: it puts Bayat Group's expert knowledge base into the conversation, cited and per-program, and it keeps working when the catalog is two hundred programs authored by providers through the Phase 7 self-service described in `docs/internal-modules/BG_MULTI_PROGRAM_STRATEGIC_FUNNEL.md`.

Both are worth building. Confusing which one solves which problem is how this ends up as a vector store bolted to a chat that still wanders.

## 6. Rollout

**Phase 0 — Index.** `pgvector` extension, `ProgramChunk` model and migration, ingestion command, reindex on save. No runtime consumer, so it is purely additive.

**Phase 1 — Shadow.** The loop runs alongside the live chat and logs the shortlist it would have produced, against what the current ranker actually produced. Nothing user-visible. This is the phase that decides whether the rest is worth finishing.

**Phase 2 — Guided flow.** The targeting loop drives the guided chat behind a funnel flag, with the phase plan as fallback. Contained and reversible.

**Phase 3 — Grounding.** Retrieved chunks and citations reach the program-locked chat and the shortlist card.

## 7. Evaluation

Roughly fifty applicant profiles labelled by a consultant with the program they should reach, run as a harness in Phase 1 before anything is user-facing.

| Metric | Baseline | Target |
|---|---|---|
| Turns to shortlist | up to 36 | ≤ 10 median |
| Top-1 agreement with consultant label | measure in shadow | ≥ 0.8 |
| Top-3 agreement | measure in shadow | ≥ 0.95 |
| Non-discriminative turns per session | measure in shadow | ≤ 1 |
| Unsourced program claims | unmeasured | 0 |

## 8. Open questions

Does the program-locked (self-select) flow adopt the loop, or only the grounding? The proposal is grounding only — its target is already chosen, so there is no candidate set to reduce.

Does the shortlist ever return a single program, or always show the runners-up? The proposal is to always show them with the margin stated, because a shortlist of one reads as a sale rather than as advice.

How is the per-signal cost calibrated? Initially by hand, from the ordering the phase plans already encode — work and business before money, family after — with the shadow run as the check.
