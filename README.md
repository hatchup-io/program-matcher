# Program Matcher

Design proposal for a system that matches an applicant to a migration **program** — and only then to a provider who can deliver it.

This repository holds the design, not the implementation. The system it describes is built in the platform backend, under the funnel-module and programs apps. The design lives here so it can be reviewed and presented before any code is written.

## The five flows

The proposal is organised around the five flows the system has to answer for, plus an action plan for each:

| # | Flow | Question it answers |
|---|---|---|
| 1 | **Collection** | How does a program get into the catalog, and what stops a wrong one getting in? |
| 2 | **Storage** | Where does program data live, and which tier is allowed to decide anything? |
| 3 | **Update** | What happens when a program changes — including mid-conversation? |
| 4 | **Client questions** | What do we ask, in what order, and when do we stop? |
| 5 | **Provider delivery** | The program is matched — now who delivers it? |

## Contents

| Path | What it is |
|---|---|
| `deck/program-matcher.md` | The presentation (Marp). Internal engineering audience. |
| `docs/ARCHITECTURE.md` | The map — the spine, the rules, and where each detail lives. |
| `docs/01-data-collection.md` | Source trust levels, the intake state machine, extraction-proposes/human-approves, the required-field contract. |
| `docs/02-data-storage.md` | The three tiers, the section taxonomy, the pre-filtered retrieval query, sizing. |
| `docs/03-data-update.md` | Revisions, the publish transaction, the drift scan, session pinning, staleness. |
| `docs/04-client-questions.md` | The loop, the signal catalog, the question bank, and five end-to-end scenarios. |
| `docs/05-provider-delivery.md` | The inversion, delivery resolution, the three cases, ordering, booking attribution. |
| `docs/06-action-plan.md` | Six workstreams broken to commit-sized tasks, with gates and acceptance criteria. |

## The proposal in three sentences

The matching conversation runs a fixed discovery script and only ranks programs once it is over, so it can spend thirty-odd turns reaching a shortlist that was settled at eight — nothing in the session knows which programs are still standing, so no question can be skipped for having stopped mattering.

We give the session an explicit candidate set, select each question by what it would rule out, stop as soon as no remaining unknown can reorder the result, and back the whole thing with a program knowledge base that is collected through a review gate, stored in three tiers, and versioned so a decision stays explicable after the source changes.

And the catalog is inverted: today programs are reached *through* a provider, which makes a well-fitting program invisible if nobody has been assigned to it — matching should end at a program, with providers resolved afterwards as the delivery layer.

## The deck

Rendered from `deck/program-matcher.md` on every push to `main` and published to GitHub Pages:

**https://hatchup-io.github.io/program-matcher/**

The same run produces `program-matcher.pdf` beside it, and attaches it to any published release. Pull requests build the deck too and leave PDF + HTML on the run as the `program-matcher-deck` artifact, so a slide change is reviewable before it reaches the URL.

Locally:

```bash
npx @marp-team/marp-cli@latest deck/program-matcher.md --html --allow-local-files -o site/index.html
npx @marp-team/marp-cli@latest deck/program-matcher.md --pdf --allow-local-files -o site/program-matcher.pdf
```

Or preview live in VS Code with the Marp extension.

## Conventions

Markdown prose is one paragraph per line, never hard-wrapped — `.markdownlint.json` disables `MD013` accordingly.
