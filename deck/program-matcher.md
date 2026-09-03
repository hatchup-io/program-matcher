---
marp: true
theme: default
paginate: true
size: 16:9
title: Program Matcher — collection, storage, update, questions, delivery
description: Design proposal — a program-first matching system, revised after review
style: |
  section {
    font-size: 24px;
    padding: 44px 58px;
  }
  section.lead {
    justify-content: center;
    text-align: left;
  }
  h1 { color: #12324f; font-size: 44px; }
  h2 { color: #12324f; font-size: 32px; border-bottom: 2px solid #e2e8f0; padding-bottom: 6px; }
  h3 { color: #0f766e; font-size: 25px; margin-bottom: 4px; }
  code { font-size: 0.82em; }
  pre { font-size: 0.64em; line-height: 1.32; background: #f7fafc; border: 1px solid #e2e8f0; }
  table { font-size: 0.74em; }
  strong { color: #0f766e; }
  blockquote { border-left: 4px solid #0f766e; color: #334155; font-style: normal; }
  footer { color: #94a3b8; font-size: 14px; }
footer: 'Program Matcher — internal engineering — v2'
---

<!-- _class: lead -->
<!-- _paginate: false -->

# Program Matcher

## Matching an applicant to a **program**

Five flows: how program data is **collected**, **stored**, **updated**; what we **ask the client**; and how **providers** are shown once a program is found.

Design proposal, v2 — revised after review — September 2026

---

## What changed in v2

Two corrections from the review, both taken:

**1. The subject is the program, not the provider.** v1 was written from inside one provider's funnel. It has been rewritten program-first; a provider is a *delivery channel* for a matched program, and any named provider in here is an example of a deployment, not the subject.

**2. Three of the five flows were missing.** v1 covered storage and part of the question logic. Collection, update and provider delivery were absent. Each now has its own section and its own line in the action plan.

| Asked for | v1 | v2 |
|---|---|---|
| Data collection flow | ✗ | §1 — intake, extraction, review gate |
| Data storage | partial | §2 — three tiers |
| Data update flow | ✗ | §3 — revisions, reindex, in-flight sessions |
| Client question scenarios | partial | §4 — signal catalog + three scenarios |
| Provider display flow | ✗ | §5 — delivery layer |
| Action plan | ✗ | §7 — six workstreams, sequenced |

---

## The correction that matters

Today the catalog is reached **through a provider**:

```
funnel ──▶ exclusive provider ──▶ their ProgramAssignment rows ──▶ programs shown
```

`CatalogService.programs_for(funnel)` returns the programs assigned to the funnel's exclusive provider. Matching therefore happens *inside* a provider's inventory, and a program nobody has assigned is invisible no matter how well it fits.

Program-first inverts it:

```
applicant ──▶ matched program(s) ──▶ providers who can deliver them ──▶ booking
```

**A provider-scoped funnel becomes a filter on delivery, not a constraint on matching.** That single change is what makes the rest of this deck coherent — and it is a real change to `CatalogService`, not a rewording.

---

## The system in one picture

```
  SOURCES                  ①  COLLECT              ②  STORE
  official briefs   ─┐                        ┌─ structured  (engines decide)
  legal references  ─┼─▶ intake → extract  ──▶├─ narrative   (chat cites)
  provider one-pagers┘    → human review      └─ provenance  (every field traceable)
                                │                        │
                          ③  UPDATE ◀───────────────────┘
                       revisions · reindex · drift scan

  applicant ──▶ ④  QUESTION LOOP ──▶ matched program(s) ──▶ ⑤  DELIVERY ──▶ booking
                candidate set +          + citations          providers who
                information gain                              can deliver it
```

Five flows, one spine. The rest of the deck walks them in order.

---

<!-- _class: lead -->

# ① Data collection

How a program gets into the catalog — and what stops a wrong one getting in.

---

## ① Where program data comes from today

A developer reads a source document and hand-writes a seed file. Seventeen programs, seventeen YAML files, one author each, no review step, no record of which sentence produced which number.

That is fine for seventeen programs written by the team that built the engine. It does not survive providers authoring their own programs, and it has no answer to *"where did this threshold come from?"* — which is the question that matters when the product quotes an investment amount or a processing time.

### The three source kinds

- **Official** — the authority's own brief, the law, the regulation. Highest trust, slowest to change, hardest to parse.
- **Expert** — the in-house methodology and advisory briefings. Trusted for *weighting and judgement*, never for legal fact.
- **Provider** — a one-pager from the provider who sells the program. Useful and current, and the one with a commercial interest — so it is the one that must be reviewed hardest.

---

## ① The intake pipeline

```
 document  ──▶  provenance stamp  ──▶  extraction  ──▶  field review  ──▶  publish
 (pdf/md)       publisher, date,       LLM proposes     human approves    revision
                language, URL/path     each field       field by field    goes live
                                       + the sentence
                                       it came from
```

**Nothing is published by extraction.** The model proposes; a person approves. Each proposed field carries the span it was drawn from, so approving is reading one sentence, not re-reading the document.

**A program cannot publish with required fields unapproved.** The intake contract is the required set — category, outcome, authority, legal basis, cost floor, processing time, family inclusion, restricted nationalities, objective ratings.

**Silence is not permission.** A source that says nothing about restricted nationalities yields *unknown*, never *unrestricted* — the same rule the current seeds already state in their own comments.

---

## ① Who does what

| Step | Owner | Gate |
|---|---|---|
| Register a source document | Program owner | Provenance fields required |
| Extract structured proposal | System (LLM) | Never auto-publishes |
| Approve fields | Domain reviewer / consultant | Per-field, with the source span visible |
| Publish revision | Program owner | Blocked if required fields unapproved |
| Provider self-authoring | Provider | Draft → publish, audited, same gates |

The panel work is real UI, not admin. Per the platform rule, an operator action that only exists in Django admin is not finished.

---

<!-- _class: lead -->

# ② Data storage

Three tiers, with one rule about which tier is allowed to decide anything.

---

## ② Three tiers

### Tier 1 — structured (relational)
`MigrationProgram`, `ProgramPlan`, eligibility parameters, objective ratings. Typed, queryable, and the **only** tier an engine reads. Eligibility and qualification compute here.

### Tier 2 — narrative (vector)
`ProgramChunk` — the source documents chunked **by section** (routes, physical presence, family inclusion, restricted nationalities, processing, citizenship pathway), embedded, indexed in `pgvector` with HNSW. This is the only tier the chat may quote from.

### Tier 3 — provenance
`ProgramSource` and `ProgramRevision` — document, publisher, date, hash, and the span each Tier-1 field was extracted from.

> **Engines read Tier 1. Language reads Tier 2. Both must resolve to Tier 3.**

---

## ② Why sections, and why pgvector

**Chunk by section, not by window.** These documents are already sectioned the way an advisor cites them. A fixed-window chunker splits a route's amount from its conditions — and that pair, separated, is exactly how a confident wrong answer is produced.

**Store vectors in the Postgres we already run:**

- **One transaction.** A program revision and its chunks commit together. No dual-write, no reconciliation job, no "the index says the route is open, the catalog says it was retired in 2023".
- **Filters and vectors in one query.** Hard constraints are SQL predicates on the same table, so they *pre-filter* rather than post-filter a top-k that was already wrong.
- **Nothing new to operate.** No extra service, no extra backup rotation, no new way for a deploy to fail.

Revisit at ~10⁵ chunks, or when a second service needs the same index.

---

## ② Honest sizing

At today's catalog size, vector search is **not** what makes the chat converge — the question loop in §4 is, and it would work over a plain queryset.

What retrieval buys is **grounding** (every claim carries a source) and **reach** (it still works at two hundred provider-authored programs).

Build both, but do not confuse them. Ship only the vector store and the chat still wanders; ship only the loop and the chat converges on something it cannot cite.

---

<!-- _class: lead -->

# ③ Data update

What happens when a program changes — including mid-conversation.

---

## ③ Revisions, not edits

A published program is **immutable**. A change creates a new `ProgramRevision`; publishing flips which revision is current, with `effective_from` / `effective_until` — the same shape `ProgramAssignment` already uses for provider bindings.

This buys three things that in-place edits cannot:

- **A decision stays explicable.** A card produced in March cites the revision it was produced from. When the threshold moves in June, the March card still reads correctly instead of quietly becoming wrong.
- **Change is reviewable.** A revision diff is what the reviewer approves, so a provider's edit to a cost floor is visible as a cost-floor change.
- **Rollback is a pointer move**, not a restore.

---

## ③ Keeping the index true

| Trigger | Action |
|---|---|
| Revision published | Re-chunk; re-embed only chunks whose `content_hash` changed |
| Seed / document changed | Same path — the seed is a source like any other |
| Manual | `manage.py reindex_programs [--program slug] [--force]` |
| Nightly | Consistency scan: every current revision has chunks, every chunk hash matches, no orphans |

The nightly scan belongs with the other periodic scans in the notifications Beat schedule. **A chunk set that has silently drifted from the catalog is worse than an error** — an error is visible; drift answers confidently and wrongly.

---

## ③ The case nobody plans for: change mid-session

An applicant is eight turns into a conversation when the program they are converging on publishes a new revision.

**A session pins the revision it started on.** The conversation, the eligibility verdict and the shortlist all read that pinned revision to the end. The next session gets the new one.

Without pinning, an applicant can be told two different things about the same program in one conversation, and the transcript will not explain why.

**Staleness is shown, not enforced.** Every program carries a `review_due` date; an overdue program is flagged in the panel and its card says *verified as of ‹date›*. It is **not** silently dropped from the shortlist — quietly shrinking the catalog is a worse failure than showing a date.

---

<!-- _class: lead -->

# ④ What we ask the client

The question loop, the signal catalog, and three scenarios end to end.

---

## ④ The loop

State on the session: the **candidate set** (which programs are still standing, and why each excluded one was excluded) alongside the signals captured so far.

```
per turn:
  1. filter    hard constraints as SQL  → exclusions get a reason code + turn number
  2. retrieve  intent → chunks → candidate programs (per-program cap)
  3. score     existing eligibility + qualification engines, unchanged
  4. select    ask the signal with the highest information gain
  5. stop?     dominance · stability · exhaustion · empty · ceiling
```

```
gain(s) = H(C) − Σ P(v)·H(C | s=v)          ask argmax gain(s)·confidence(s)/cost(s)
```

**A question whose answer cannot reorder the candidate set is never asked.** That one rule is what replaces a fixed script that runs to its turn budget whether or not the answer is already known.

---

## ④ The signal catalog

| Signal | Source | Cost | What it splits |
|---|---|---|---|
| Primary objective | Chat, turn 1 | Low | Citizenship vs residence vs visa — the widest cut available |
| Nationality | Profile / form | Free | Hard gate: restricted lists |
| Budget band | Chat | Medium | Cost floors — the sharpest numeric split |
| Timeline | Chat / form | Low | Processing times |
| Family composition | Form + chat | Low | Plan tier, dependent age limits |
| Physical presence tolerance | Chat | Medium | Separates residence programs from each other |
| Source of funds | Chat | **High** | Rarely splits; asked late, or only when decisive |

**Cost is not difficulty — it is what the question costs the applicant.** A passport question is cheap; an undisclosed-funds question is expensive and is earned, not opened with.

---

## ④ Scenario A — the clean convergence

Clear objective, budget-bound, no blocks.

| Turn | Captured | Candidates | Why this question |
|---|---|---|---|
| 1 | Objective = residence for the family | 17 → 8 | Widest cut at cold start |
| 2 | Nationality | 8 → 5 | Hard gate; three excluded **with a stated reason** |
| 3 | Budget band | 5 → 3 | Sharpest remaining numeric split |
| 4 | Family = spouse + 2 children | 3 → 3 | No reordering — but sets the plan tier |
| 5 | Timeline ≤ 12 months | 3 → 2 | Processing time separates the leaders |

**Stops at 5.** Two programs presented with citations and the margin between them; everything else the file needs is collected by the roadmap form, not by the chat.

---

## ④ Scenarios B and C — the ones that go wrong

### B — hard block on turn 2
Nationality is on a restricted list for most of the surviving set. The loop does **not** keep interviewing: it says which programs are closed and why, presents whatever survives, and routes to a consultant if nothing does. A mitigation probe (a second nationality, for instance) is raised **immediately**, not saved for the end — asking eight more questions before delivering bad news is the worst version of this conversation.

### C — contradiction
Turn 3 answer contradicts turn 1 ("residence for the family" then "I only travel for business"). The loop reopens the affected signal rather than averaging them, and asks one disambiguating question naming both answers. If it stays ambiguous, both readings stay in the candidate set and the shortlist ships with a **near-miss tier** rather than a false confidence.

**Never re-ask what is already known or derivable.** A stated family total that is fully accounted for closes the dependent questions; a stated purpose that is a permanent move answers the intent question.

---

## ④ What the model does and does not do

**Does:** run the conversation, extract signals from natural language, phrase the selected question, present the result in the right voice for the category.

**Does not:** choose the candidate set, score eligibility, compute qualification, order the shortlist, or state a program fact that is not in a retrieved chunk.

Grounding is enforced at the prompt boundary: the turn prompt receives the retrieved chunks for surviving candidates and no other program content, and the guardrail pass flags any program claim without a supporting chunk.

> Retrieval narrows. Deterministic engines decide. The model only speaks.

---

<!-- _class: lead -->

# ⑤ Provider display

The program is matched. Now — who delivers it?

---

## ⑤ The delivery layer

Matching ends at a program. Delivery begins there:

```
matched program
      │
      ▼
 currently-effective ProgramAssignment rows        ← who may deliver this program
      │
      ├── exclusive assignment  ──▶ one provider, no choice (that is what exclusive means)
      └── non-exclusive         ──▶ several providers ──▶ ordered, with the rule stated
      │
      ▼
 provider card: fee, timeline, languages, track record, exclusivity
      │
      ▼
 that provider's consultants ──▶ timeslot ──▶ booking (program + plan + assignment stamped)
```

The booking already carries program, plan and assignment for attribution. What changes is where they come from: **the matched program**, rather than the funnel's provider.

---

## ⑤ The three cases

**One exclusive provider.** No choice is presented, because there is none. The card says so plainly rather than implying a shortlist of one was a selection.

**Several providers.** The ordering rule must be **explicit and auditable** — not whatever the queryset happened to return. Candidate inputs: rating, capacity, response time, commercial terms. This is a business decision and it is listed in §8 as one we need from you.

**No provider.** The program is still shown, marked as *not currently deliverable*, and routed to a consultant. Hiding a program because nobody has been assigned to it is how a good match disappears silently — the failure v1's provider-first catalog had by construction.

**A provider-scoped funnel filters this layer.** It no longer decides what can be matched.

---

## ⑥ What we build

```python
class ProgramSource(TimeStampedModel):        # tier 3
    publisher, document_date, kind, uri, content_hash

class ProgramRevision(TimeStampedModel):      # tier 3
    program, source, status(draft|published), effective_from, effective_until
    approved_fields  # per-field approval + the source span

class ProgramChunk(TimeStampedModel):         # tier 2
    revision, section_type, content, content_hash, embedding(3072)
    # HNSW on embedding; btree on (revision_id, section_type)
```

On the chat session, alongside the existing signal state:

- `pinned_revision` — the revision the session reads to the end.
- `candidate_state` — surviving programs with scores, plus every exclusion with its reason code and turn. **This is the audit trail for "why this shortlist"**, and the thing that makes the loop debuggable in the operator panel.

---

## ⑦ Action plan

| # | Workstream | Depends on | Size | Output |
|---|---|---|---|---|
| **W1** | Storage tiers: revisions, sources, chunks, `pgvector` | — | 1.5 wk | Migrations + reindex command |
| **W2** | Collection: intake, extraction proposal, review panel | W1 | 2.5 wk | A program can be authored and approved end to end |
| **W3** | Update: publish/rollback, drift scan, session pinning | W1 | 1 wk | Index provably matches the catalog |
| **W4** | Question loop: candidate state, signal catalog, gain selection | W1 | 2 wk | Behind a flag, shadow first |
| **W5** | Delivery layer: program → providers → booking attribution | — | 1.5 wk | `CatalogService` inverted |
| **W6** | Evaluation harness + golden set | W4 | 1 wk | The numbers that decide W4 ships |

**Sequencing.** W1 first and alone. Then W2, W3, W4 in parallel. **W5 has no dependency on any of them and can start immediately** — it is the change that makes the system program-first, and it is the cheapest item on this list.

---

## ⑦ Sequence

```
week 1        2         3         4         5         6
 ├─ W1 ────────┤
 │             ├─ W2 ──────────────────────┤
 │             ├─ W3 ──┤
 │             ├─ W4 ─────────────┤
 │                                ├─ W6 ───┤   (shadow → decide)
 ├─ W5 ─────────────┤                          (independent, start now)
```

**Two decision points, not one deploy.** End of week 3: does the intake gate hold when a real program goes through it? End of week 6: does the loop beat the current ranker on the golden set? Nothing user-facing ships before the second one.

---

## ⑧ Evaluation — or it is just an opinion

Roughly fifty applicant profiles labelled by a consultant with the program they should reach. The harness lands in W6, before anything is user-facing.

| Metric | Baseline | Target |
|---|---|---|
| Turns to shortlist | up to 36 | ≤ 10 median |
| Top-1 agreement with the consultant label | measure in shadow | ≥ 0.8 |
| Top-3 agreement | measure in shadow | ≥ 0.95 |
| Non-discriminative turns per session | measure in shadow | ≤ 1 |
| Program claims with no source | unmeasured | 0 |
| Index-vs-catalog drift | unmeasured | 0 |

"The chat feels more directed" is not a result.

---

## ⑧ Decisions we need from you

1. **Provider ordering** for a non-exclusive program — rating, capacity, response time, or commercial terms? The rule has to be stated somewhere, and it should be stated by the business rather than emerge from a queryset.
2. **Who approves a field** at intake — an internal domain reviewer, or the provider for their own programs with an internal spot-check?
3. **A consultant's time for the golden set.** Fifty labelled profiles is the difference between a measured improvement and a hunch.
4. **Review cadence** — how stale is too stale before a program is flagged?

### Open risks
Extraction quality at intake (mitigated by per-field approval, measured by approval-edit rate) · retrieval bias toward verbose documents (per-program cap; verify in shadow) · over-pruning a program a consultant would have kept (every exclusion carries a reason; near-miss tier).

Spec and open threads: `docs/ARCHITECTURE.md` in this repository.
