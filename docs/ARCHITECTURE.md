# Program Matcher — architecture

> Status: proposal, v2. Nothing here is built. The deck in `deck/program-matcher.md` is the summary; this file is the detail behind it.

The subject of this design is the **program**. A provider is a delivery channel for a program that has already been matched, and a provider-scoped funnel is a filter on delivery — not a constraint on what can be matched. Where a specific deployment is named below it is an example of the shape, not the subject.

The implementation target is the platform backend, in the funnel-module and programs apps. This repository holds the design so it can be reviewed and presented before code exists.

## 0. What v2 changed

v1 was written from inside a single provider's funnel, and covered only data storage plus part of the question logic. The review asked for five flows and an action plan. v2 restructures around them: collection (§1), storage (§2), update (§3), client questions (§4), provider delivery (§5), with the action plan in §7.

## 1. Data collection

### 1.1 Where the data comes from today

A developer reads a source document and hand-writes a seed YAML file; seventeen programs, one author each, no review step, and no record of which sentence in the source produced which number in the seed.

That is workable while the team that built the engine also authors the catalog. It does not survive providers authoring their own programs, and it has no answer to the question that matters most in a regulated advisory product: *where did this threshold come from?*

### 1.2 Three kinds of source, three levels of trust

**Official** sources — the authority's own brief, the law, the regulation — carry the highest trust, change slowest, and are the hardest to parse. **Expert** sources — in-house methodology and advisory briefings — are trusted for weighting and judgement, never for legal fact. **Provider** sources — the one-pager from whoever sells the program — are useful and current, and carry a commercial interest, which makes them the ones to review hardest.

### 1.3 The pipeline

A document enters with a mandatory provenance stamp: publisher, document date, language, and a URI or repository path. An extraction pass then proposes the structured block — facts, plans, eligibility parameters, objective ratings — and the narrative sections, with **every proposed field carrying the source span it was drawn from**.

Nothing is published by extraction. A human approves field by field, and approving is reading one sentence rather than re-reading the document. A program cannot publish while a required field is unapproved; the required set is the intake contract — category, outcome, authority, legal basis, cost floor, processing time, family inclusion, restricted nationalities, objective ratings.

**Silence is not permission.** A source that says nothing about restricted nationalities yields *unknown*, never *unrestricted*. The current seeds already state this rule in their own comments; intake makes it structural.

### 1.4 Ownership

The program owner registers sources and publishes revisions. A domain reviewer or consultant approves fields. The system only ever proposes. Providers self-author their own programs through the same gates, draft to publish, audited.

The review surface is real panel UI. Per the platform rule, an operator action that exists only in Django admin is not finished.

## 2. Data storage

### 2.1 Three tiers

**Tier 1, structured (relational).** Programs, plans, eligibility parameters, objective ratings. Typed and queryable, and the only tier an engine reads: eligibility and qualification compute here.

**Tier 2, narrative (vector).** Source documents chunked by section — qualifying routes, physical presence, family inclusion, restricted nationalities, processing, citizenship pathway — embedded and indexed in `pgvector` with an HNSW index. The only tier the chat may quote from.

**Tier 3, provenance.** Sources and revisions: document, publisher, date, hash, and the span each Tier-1 field was extracted from.

The rule that keeps the tiers honest: **engines read Tier 1, language reads Tier 2, and both must resolve to Tier 3.**

### 2.2 Chunking by section

The documents are already sectioned the way an advisor cites them, and a section is the unit a claim is made from. A fixed-window chunker splits a route's amount from its conditions, and that pair — separated — is precisely how a confident wrong answer gets produced.

### 2.3 Why pgvector rather than a dedicated vector database

One store means a program revision and its chunks commit in the same transaction: no dual-write, no reconciliation job, and no state where the index still offers a route the catalog retired. Filters and vectors live in one query, so hard constraints pre-filter rather than post-filter a top-k that was already wrong. And nothing new joins the deployment — no extra service, no extra backup rotation, no new way for a deploy to fail.

Revisit at roughly 10⁵ chunks, or as soon as a second service needs the same index.

### 2.4 Honest sizing

At today's catalog size, vector search is not what makes the chat converge — the question loop in §4 is, and it would work over a plain queryset. What retrieval buys is grounding, so every claim carries a source, and reach, so the same design still works at two hundred provider-authored programs.

Both are worth building. Confusing which one solves which problem is how this ends up as a vector store bolted onto a chat that still wanders.

## 3. Data update

### 3.1 Revisions, not edits

A published program is immutable. A change creates a new revision; publishing flips which revision is current, bounded by `effective_from` and `effective_until` — the same shape provider assignments already use.

This buys three things in-place edits cannot. A decision stays explicable, because a card cites the revision it was produced from and stays correct after the threshold moves. Change is reviewable, because a revision diff is what the reviewer approves. And rollback is a pointer move rather than a restore.

### 3.2 Keeping the index true

Publishing a revision re-chunks it and re-embeds only the chunks whose content hash changed. A changed seed or document takes the same path, because a seed is a source like any other. `manage.py reindex_programs [--program slug] [--force]` covers manual repair, and a nightly consistency scan — alongside the existing periodic scans — asserts that every current revision has chunks, every chunk hash matches, and no orphans survive.

A chunk set that has silently drifted from the catalog is worse than an error. An error is visible; drift answers confidently and wrongly.

### 3.3 Change mid-session

A session pins the revision it started on, and the conversation, the eligibility verdict and the shortlist all read that pinned revision through to the end. The next session gets the new one.

Without pinning, an applicant can be told two different things about the same program inside one conversation, and the transcript will not explain why.

### 3.4 Staleness

Every program carries a review-due date. An overdue program is flagged in the panel and its card states *verified as of* the source date. It is not silently dropped from the shortlist: quietly shrinking the catalog is a worse failure than showing a date.

## 4. Client questions

### 4.1 The loop

The session carries a candidate set — which programs are still standing, and why each excluded one was excluded — alongside the signals captured so far.

Each turn: apply captured hard constraints as SQL predicates, recording every exclusion with a reason code and the turn number; retrieve over the surviving programs' chunks with a per-program cap so a long document cannot outvote a terse one; score the candidates through the existing eligibility and qualification engines, unchanged; select the next question by expected information gain; then test the stop conditions before generating anything.

```
gain(s) = H(C) − Σ P(v)·H(C | s=v)        ask argmax gain(s)·confidence(s)/cost(s)
```

A question whose answer cannot reorder the candidate set has zero gain and is never asked. That single rule is what replaces a fixed script that runs to its turn budget whether or not the answer is already known.

Stop conditions: dominance (the leader's margin exceeds what any remaining unknown can close), stability (top-k unchanged for *n* turns with only non-discriminative signals open), exhaustion (no positive-gain signal remains), empty (nothing survives the hard gates — presented honestly and routed to a consultant), and a hard turn ceiling as a backstop rather than as the mechanism.

### 4.2 Signal cost

Cost is not difficulty; it is what the question costs the applicant. Nationality and primary objective are cheap. Budget band is moderate. Source of funds is expensive, rarely splits the candidate set, and is therefore asked late or only when it is decisive. The initial ordering comes from the sequence the current phase plans already encode — work and business before money, family after — and is checked against the shadow run.

### 4.3 Scenarios

**Clean convergence.** Objective, nationality, budget, family, timeline — five turns, two survivors presented with citations and the margin between them. Whatever the file still needs is collected by the roadmap form rather than by the chat.

**Hard block.** When a restricted-nationality gate closes most of the surviving set, the loop stops interviewing: it names what is closed and why, presents whatever survives, and routes to a consultant if nothing does. A mitigation probe is raised immediately rather than saved for the end — asking eight more questions before delivering bad news is the worst version of this conversation.

**Contradiction.** When a later answer contradicts an earlier one, the loop reopens the affected signal instead of averaging the two, and asks one disambiguating question that names both answers. If it stays ambiguous, both readings stay in the candidate set and the shortlist ships with a near-miss tier rather than a false confidence.

Across all three: never re-ask what is already known or derivable. A family total that is fully accounted for closes the dependent questions; a stated purpose that is a permanent move answers the intent question.

### 4.4 The model boundary

The model runs the conversation, extracts signals, phrases the selected question, and presents the result in the right voice for the category. It does not choose the candidate set, score eligibility, compute qualification, order the shortlist, or assert a program fact that is not in a retrieved chunk.

Grounding is enforced at the prompt boundary: the turn prompt receives the retrieved chunks for surviving candidates and no other program content, and the guardrail pass flags a program claim with no supporting chunk the same way it already screens input. The shortlist card carries a sources block — document and section — so a consultant reading a transcript can verify a number against the original file.

## 5. Provider delivery

### 5.1 The inversion

Today the catalog is reached through a provider: a funnel resolves to its exclusive provider, and the programs shown are that provider's currently-effective assignments. Matching therefore happens inside one provider's inventory, and a program nobody has assigned is invisible however well it fits.

Program-first reverses the order. The applicant is matched to a program; the delivery layer then resolves which providers hold a currently-effective assignment for it, presents them, and carries the choice into a booking. A provider-scoped funnel filters that layer rather than deciding what could be matched. This is a real change to the catalog service, not a rewording.

### 5.2 The three cases

**One exclusive provider.** No choice is presented, because there is none. The card says so plainly instead of implying that a shortlist of one was a selection.

**Several providers.** The ordering rule must be explicit and auditable rather than whatever the queryset happened to return. Candidate inputs are rating, capacity, response time and commercial terms; the choice is a business decision, listed in §8.

**No provider.** The program is still shown, marked as not currently deliverable, and routed to a consultant. Hiding a program because nobody has been assigned to it is exactly the silent failure the provider-first catalog has by construction.

### 5.3 Booking attribution

A booking already stamps program, plan and assignment. What changes is where they come from: the matched program, rather than the funnel's exclusive provider — which also removes the current restriction that attribution only fills in for exclusive funnels holding a single-candidate card.

## 6. What we build

```python
class ProgramSource(TimeStampedModel):        # tier 3
    publisher, document_date, kind, uri, content_hash

class ProgramRevision(TimeStampedModel):      # tier 3
    program, source, status(draft|published), effective_from, effective_until
    approved_fields   # per-field approval + the source span it came from

class ProgramChunk(TimeStampedModel):         # tier 2
    revision, section_type, content, content_hash, embedding(3072)
    # HNSW on embedding; btree on (revision_id, section_type)
```

On the chat session, alongside the existing signal state: `pinned_revision`, the revision the session reads to the end; and `candidate_state`, the surviving programs with their scores plus every exclusion with its reason code and turn number. The latter is the audit trail for *why this shortlist*, and the thing that makes the loop debuggable from the operator panel.

## 7. Action plan

| # | Workstream | Depends on | Size | Output |
|---|---|---|---|---|
| W1 | Storage tiers: revisions, sources, chunks, `pgvector` | — | 1.5 wk | Migrations + reindex command |
| W2 | Collection: intake, extraction proposal, review panel | W1 | 2.5 wk | A program authored and approved end to end |
| W3 | Update: publish/rollback, drift scan, session pinning | W1 | 1 wk | Index provably matches the catalog |
| W4 | Question loop: candidate state, signal catalog, gain selection | W1 | 2 wk | Behind a flag, shadow first |
| W5 | Delivery layer: program → providers → booking attribution | — | 1.5 wk | Catalog service inverted |
| W6 | Evaluation harness + golden set | W4 | 1 wk | The numbers that decide whether W4 ships |

W1 runs first and alone. W2, W3 and W4 then run in parallel. **W5 depends on nothing here and can start immediately** — it is the change that makes the system program-first, and it is the cheapest item on the list.

There are two decision points rather than one deploy. At the end of week 3: does the intake gate hold when a real program goes through it? At the end of week 6: does the loop beat the current ranker on the golden set? Nothing user-facing ships before the second.

## 8. Decisions needed, and open risks

**Provider ordering** for a non-exclusive program — rating, capacity, response time, or commercial terms. The rule has to be stated by the business rather than emerge from a queryset.

**Field approval ownership** at intake — an internal domain reviewer, or the provider for their own programs with an internal spot-check.

**Consultant time for the golden set** — roughly fifty labelled profiles.

**Review cadence** — how stale a source may be before its program is flagged.

Open risks: extraction quality at intake, mitigated by per-field approval and measured by the approval-edit rate; retrieval bias toward verbose documents, capped per program and verified in shadow; over-pruning a program a consultant would have kept, mitigated by reason codes on every exclusion and a near-miss tier on the shortlist.

## 9. Evaluation

| Metric | Baseline | Target |
|---|---|---|
| Turns to shortlist | up to 36 | ≤ 10 median |
| Top-1 agreement with consultant label | measure in shadow | ≥ 0.8 |
| Top-3 agreement | measure in shadow | ≥ 0.95 |
| Non-discriminative turns per session | measure in shadow | ≤ 1 |
| Program claims with no source | unmeasured | 0 |
| Index-vs-catalog drift | unmeasured | 0 |
