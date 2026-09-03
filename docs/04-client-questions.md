# ④ Client questions — what we ask, in what order, and when we stop

> Answers: *سناریوهای سوالات از کلاینت‌ها*. Sibling docs: [collection](01-data-collection.md) · [storage](02-data-storage.md) · [update](03-data-update.md) · [delivery](05-provider-delivery.md) · [action plan](06-action-plan.md)

## 4.1 What is wrong with the current conversation

The chat runs a fixed plan: six phases, each with a turn budget, advancing when its required signals arrive **or** when the budget is spent. Programs are ranked afterwards.

Nothing in the session knows which programs are still standing. So no question can be skipped on the grounds that its answer no longer changes anything, and a conversation can spend thirty-odd turns arriving at a shortlist that was settled at eight. The applicant experiences this as an interview; the business experiences it as drop-off.

## 4.2 The loop

The session carries a **candidate set** alongside its signals: which programs are still viable, and why each excluded one was excluded.

```python
candidate_state = {
  "surviving": {
    "portugal_rbi": {"score": 0.81, "verdict": "ELIGIBLE",  "band": "qualified"},
    "greece_rbi":   {"score": 0.74, "verdict": "ELIGIBLE",  "band": "qualified"},
  },
  "excluded": {
    "turkiye_cbi":  {"code": "objective_mismatch",    "turn": 1},
    "nauru_cbi":    {"code": "nationality_sanctioned","turn": 2},
    "malta_rbi":    {"code": "below_contribution_min","turn": 3},
  },
  "asked": ["primary_objective", "nationality", "budget_band"],
  "pinned_revision_ids": {"portugal_rbi": 41, "greece_rbi": 38},
}
```

Per turn, in order:

```
1. FILTER     apply captured hard constraints as SQL predicates
              every exclusion records a reason code and the turn number
2. RETRIEVE   embed the accumulated intent; search surviving programs' chunks
              per-program cap, so a long document cannot outvote a terse one
3. SCORE      run candidates through the existing eligibility + qualification
              engines — unchanged code, unchanged verdicts
4. SELECT     ask the unanswered signal with the highest expected information gain
5. STOP?      test the stop conditions BEFORE generating another question
```

## 4.3 Question selection

```
gain(s) = H(C) − Σ  P(v) · H(C | s = v)
                v ∈ values(s)

ask = argmax   gain(s) · confidence(s) / cost(s)
       s ∈ open
```

```python
def next_question(candidates, signals, bank):
    open_signals = [s for s in bank if s.key not in signals]
    scored = []
    for s in open_signals:
        partitions = [subset(candidates, s, v) for v in s.values]
        g = entropy(candidates) - sum(
            (len(p) / len(candidates)) * entropy(p) for p in partitions
        )
        if g <= 0:
            continue                      # cannot reorder anything — never asked
        scored.append((g * s.confidence / s.cost, s))
    if not scored:
        return None                       # exhaustion: stop, do not fill time
    return max(scored)[1]
```

**A question whose answer cannot reorder the candidate set is never asked.** That single rule is what replaces a script that runs to its budget whether or not the answer is already known.

### Cost is what the question costs the applicant

Not how hard it is to answer — how much it costs to be asked. A passport question is cheap. A question about undisclosed funds is expensive, rarely splits the candidate set, and is therefore asked late or only when it is decisive. Opening with it is how a consultation feels like an interrogation.

The initial cost ordering comes from the sequence the current phase plans already encode — work and business before money, family after — and is checked against the shadow run.

## 4.4 The signal catalog

| Signal | Type | Source | Cost | What it splits |
|---|---|---|---|---|
| `primary_objective` | enum | Chat, turn 1 | 1 | Citizenship vs residence vs visa — the widest cut available |
| `optimization_priorities` | multi | Chat, turn 1 | 1 | Feeds the objective ranker |
| `nationality` | country | Profile / form | 0 | **Hard gate** — restricted lists |
| `country_of_residence` | country | Profile / form | 0 | Hard gate on some routes |
| `budget_band` | enum | Chat | 3 | Cost floors — the sharpest numeric split |
| `desired_timeline` | enum | Chat / form | 2 | Processing times |
| `family_composition` | struct | Form + chat | 2 | Plan tier, dependent age limits |
| `dependent_ages` | list | Chat | 2 | Age-limit gates |
| `physical_presence_tolerance` | enum | Chat | 3 | Separates residence programs from each other |
| `mobility_intent` | enum | Chat | 2 | Genuine-visitor rules on immigration routes |
| `business_status` | enum | Chat | 3 | Business and founder routes |
| `prior_refusals` | bool | Chat | 5 | Rarely splits; matters when it does |
| `source_of_funds_clarity` | enum | Chat | 8 | Qualification score, not eligibility — asked last |

Signals marked source **Profile / form** cost zero because they are already known — and a known signal is never asked. The single most common complaint about the current chat is being asked something the form already collected.

## 4.5 The question bank

Each signal owns its wording, its option labels, and its fallbacks — authored, not improvised, so that the *selection* is dynamic while the *phrasing* is reviewed.

```python
Signal(
  key="primary_objective",
  type="enum",
  values=("citizenship", "residency", "global_expansion", "asset_protection",
          "tax_optimization", "family_relocation", "education"),
  cost=1,
  ask={
    "en": "Before we talk about any country — what are you actually trying to achieve?",
    "fa": "قبل از اینکه دربارهٔ کشوری صحبت کنیم — دقیقاً به دنبال چه چیزی هستید؟",
  },
  labels={
    "residency":         {"en": "The right to live somewhere", "fa": "حق زندگی در یک کشور"},
    "citizenship":       {"en": "A second passport",           "fa": "پاسپورت دوم"},
    "family_relocation": {"en": "Relocating my family",        "fa": "مهاجرت خانواده‌ام"},
    "asset_protection":  {"en": "Protecting my assets",        "fa": "حفاظت از دارایی‌هایم"},
    # ...
  },
)
```

Three rules the bank is held to, all of which the current implementation already enforces for its own labels:

**Every option-typed question ships `en` and `fa` labels**, and completeness is enforced by test — a new option cannot ship without them. A question over a finite answer set is never rendered without chips, because a chipless enum question in a non-English conversation is how extraction fails.

**Extracted values stay canonical English tokens** whatever language the conversation runs in. The applicant taps *پاسپورت دوم*; the signal stores `second_passport`.

**Related questions may be asked together.** A signal group — objective, priority and the reason behind them — is invited in one breath and followed up only on what is still missing. Walking overlapping targets one at a time reads as re-asking the same question three ways.

## 4.6 Stop conditions

| Condition | Test |
|---|---|
| **Dominance** | The leader's margin exceeds what any remaining open signal could close |
| **Stability** | Top-k unchanged for *n* turns and only non-discriminative signals remain |
| **Exhaustion** | No open signal has positive gain |
| **Empty** | Nothing survives the hard gates — present honestly, route to a consultant |
| **Ceiling** | A hard turn cap, kept as a backstop rather than as the mechanism |

Whatever the file still needs but the loop never asked is collected by the roadmap form afterwards. **The conversation's job is to find the program, not to complete the file.**

## 4.7 Scenarios

### A — Clean convergence

Objective clear, budget-bound, no blocks.

| Turn | Asked | Captured | Candidates | Why |
|---|---|---|---|---|
| 1 | What are you trying to achieve? | `residency`, family-led | 17 → 8 | Widest cut available |
| 2 | *(not asked — from profile)* | `nationality` | 8 → 5 | Hard gate; 3 excluded with reasons |
| 3 | What are you comfortable committing? | `250–500k` | 5 → 3 | Sharpest numeric split |
| 4 | Who is coming with you? | spouse + 2 children | 3 → 3 | No reordering — sets the plan tier |
| 5 | What is your timeline? | `≤ 12 months` | 3 → 2 | Processing time separates the leaders |

**Stops at 5.** Two programs, each with citations and the margin between them stated.

### B — Hard block on turn 2

The applicant's nationality is on the restricted list of most of the surviving set.

> **System:** Two of the routes we were looking at don't accept applications from your country of nationality — that is their rule, not an assessment of you. One route still does, and there is a second possibility if you hold, or could hold, another nationality. Do you?

The loop does **not** keep interviewing. It names what is closed and why, presents what survives, and routes to a consultant if nothing does. The mitigation probe — a second nationality, or funds held elsewhere — is raised **immediately**, not saved for the end.

Asking eight more questions before delivering bad news is the worst version of this conversation, and it is the version a fixed script produces by construction.

### C — Contradiction

Turn 1: *"I want to move my family to Europe."* Turn 4: *"I travel for business, I wouldn't actually live there."*

> **System:** Let me make sure I have this right — earlier you mentioned moving the family, and just now that you would not be living there yourself. Which is closer: the family settles and you travel, or nobody relocates and this is about access?

The loop **reopens** the affected signal instead of averaging the two readings. If it stays ambiguous, both readings stay in the candidate set and the shortlist ships with a **near-miss tier** rather than a false confidence.

### D — Refusal or evasion

The applicant declines to give a budget band.

The signal is marked `refused` rather than left open, so it is not asked again. The loop continues on the signals that remain discriminative, and the shortlist is delivered with a stated caveat: *ranked without a budget; cost floors range from X to Y across these programs.*

A refused answer is information. Re-asking it is how a conversation becomes an interrogation.

### E — The applicant asks a question back

> **Applicant:** How long does the Portuguese one actually take?

The system answers **from retrieved chunks only**, with the source, and then returns to the selected question rather than dropping its thread:

> **System:** Around 12–18 months, and the source notes a backlog at the authority — that is from their August 2026 brief. On the timeline side, is twelve months a hard constraint for you, or a preference?

Answering is not a detour: it is the applicant handing over intent for free. Refusing to answer, or answering from the model's own knowledge, are the two failure modes — the first wastes the signal, the second invents a number.

## 4.8 Anti-patterns

| Anti-pattern | Rule |
|---|---|
| Asking what the form already answered | A known or derivable signal is never asked |
| Asking what cannot change the outcome | Zero gain, never asked |
| Averaging a contradiction | Reopen and disambiguate, naming both answers |
| Burying bad news | A hard block is delivered the turn it is discovered |
| Re-asking a refusal | `refused` is a terminal state for that signal |
| A chipless enum question | Deterministic label fallback, `en` + `fa` guaranteed |
| Naming a fact with no chunk | Guardrail rejects the turn |

**Derivation, not interrogation.** A family total that is fully accounted for closes the dependent questions. A stated purpose that is a permanent move answers the intent question. A business status that is not *not an owner* answers the ownership question. The current implementation already derives all three; the loop keeps them and adds the candidate set as a second source of skipping.

## 4.9 The model boundary

**The model does:** run the conversation, extract signals from natural language, phrase the selected question, present the result in the right voice for the category.

**The model does not:** choose the candidate set, score eligibility, compute qualification, order the shortlist, or state a program fact that is not in a retrieved chunk.

The turn prompt receives the retrieved chunks for surviving candidates and no other program content. The guardrail pass flags a program claim with no supporting chunk, the same way it already screens input. The shortlist card carries a sources block — document and section — so a consultant reading a transcript can verify a number against the original file.

## 4.10 Acceptance criteria

A signal already present in the profile or form is never asked. A signal with zero information gain is never asked. A hard block is surfaced in the turn it is detected. A refused signal is not re-asked. Every option-typed question renders chips in `en` and `fa`. Every program claim in a transcript resolves to a chunk. The candidate state records a reason code and turn number for every exclusion. Median turns-to-shortlist ≤ 10 on the golden set.

## 4.11 Open decisions

Does the program-locked flow — where the applicant has already chosen a program — adopt the loop? Recommendation: no. Its target is chosen, so there is no candidate set to reduce; it takes the grounding and citations only.

Does the shortlist ever return a single program? Recommendation: no — always show the runners-up with the margin stated. A shortlist of one reads as a sale rather than as advice.
