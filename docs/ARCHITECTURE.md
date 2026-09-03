# Program Matcher — architecture

> Status: proposal, v3. Nothing here is built. The deck in `../deck/program-matcher.md` is the presentation; this file is the map; the numbered docs are the detail.

## The subject is the program

A provider is a delivery channel for a program that has already been matched, and a provider-scoped funnel is a filter on delivery — not a constraint on what can be matched. Where a specific deployment is named anywhere in these documents it is an example of the shape, not the subject.

This is not only framing. The catalog today is reached *through* a provider's assignments, which means a program nobody has been assigned to is invisible however well it fits. Inverting that is [workstream W5](06-action-plan.md), and it is the cheapest change in the plan.

## The five flows

| # | Flow | Answers | Detail |
|---|---|---|---|
| ① | Collection | How does a program enter the catalog, and what stops a wrong one getting in? | [01-data-collection.md](01-data-collection.md) |
| ② | Storage | Where does program data live, and which tier may decide anything? | [02-data-storage.md](02-data-storage.md) |
| ③ | Update | What happens when a program changes — including mid-conversation? | [03-data-update.md](03-data-update.md) |
| ④ | Client questions | What do we ask, in what order, and when do we stop? | [04-client-questions.md](04-client-questions.md) |
| ⑤ | Provider delivery | The program is matched — now who delivers it? | [05-provider-delivery.md](05-provider-delivery.md) |
| — | Action plan | Who builds what, in what order, and what gates it? | [06-action-plan.md](06-action-plan.md) |

## The spine

```
  SOURCES                  ①  COLLECT                    ②  STORE
  official briefs   ─┐                                ┌─ tier 1  structured  (engines decide)
  legal references  ─┼─▶ intake → extract → review ──▶├─ tier 2  narrative   (chat cites)
  provider one-pagers┘    provenance   propose  approve└─ tier 3  provenance (everything resolves here)
                                            │                        │
                                      ③  UPDATE ◀───────────────────┘
                                   revisions · hash-diff reindex
                                   drift scan · session pinning

  applicant ──▶ ④  QUESTION LOOP ──▶ matched program(s) ──▶ ⑤  DELIVERY ──▶ booking
                candidate set +           + citations           providers who
                information gain                                can deliver it
```

## The three rules that hold it together

**Engines read Tier 1. Language reads Tier 2. Both must resolve to Tier 3.** A verdict computed from prose is a wrong verdict that reads convincingly; a sentence quoted from a structured field is a sentence with no source.

**A question whose answer cannot reorder the candidate set is never asked.** This is what replaces a fixed script that runs to its turn budget whether or not the answer is already known.

**Silence is not permission.** A source that says nothing about a restriction yields *unknown*, never *unrestricted* — at intake, in the verdict, and on the card.

## What the model does, and does not

**Does:** run the conversation, extract signals from natural language, phrase the selected question, propose structured fields at intake, present results in the right voice.

**Does not:** choose the candidate set, score eligibility, compute qualification, order the shortlist, publish a field, or state a program fact that is not in a retrieved chunk.

> Retrieval narrows. Deterministic engines decide. The model only speaks — and only from a source.

## Honest sizing

At today's catalog size, vector search is not what makes the conversation converge — the question loop is, and it would work over a plain queryset. What retrieval buys is grounding, so every claim carries a source, and reach, so the same design still works at two hundred provider-authored programs.

Both are worth building. Confusing which one solves which problem is how this ends up as a vector store bolted onto a conversation that still wanders.

## Version history

**v3** — each of the five flows expanded into its own specification; action plan broken down to commit-sized tasks with gates, acceptance criteria and the business decisions each one blocks on.

**v2** — reframed program-first after review; added the three flows v1 omitted (collection, update, provider delivery) and a first action plan.

**v1** — the question loop and the vector index, written from inside a single provider's funnel.
