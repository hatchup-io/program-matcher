# program-matcher

Design proposal for a system that matches an applicant to a migration **program** — and only then to a provider who can deliver it.

This repository holds the design, not the implementation. The system it describes is built in the platform backend, under the funnel-module and programs apps. The design lives here so it can be reviewed and presented before any code is written. Its audience is the engineering team. The only thing that runs is the deck, which GitHub Pages serves at <https://hatchup-io.github.io/program-matcher/>. The repository is public.

## What is inside

### The five flows

The proposal is organised around the five flows the system has to answer for, plus an action plan for each:

| # | Flow | Question it answers |
|---|---|---|
| 1 | **Collection** | How does a program get into the catalog, and what stops a wrong one getting in? |
| 2 | **Storage** | Where does program data live, and which tier is allowed to decide anything? |
| 3 | **Update** | What happens when a program changes — including mid-conversation? |
| 4 | **Client questions** | What do we ask, in what order, and when do we stop? |
| 5 | **Provider delivery** | The program is matched — now who delivers it? |

### Contents

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

### The proposal in three sentences

The matching conversation runs a fixed discovery script and only ranks programs once it is over, so it can spend thirty-odd turns reaching a shortlist that was settled at eight — nothing in the session knows which programs are still standing, so no question can be skipped for having stopped mattering.

We give the session an explicit candidate set, select each question by what it would rule out, stop as soon as no remaining unknown can reorder the result, and back the whole thing with a program knowledge base that is collected through a review gate, stored in three tiers, and versioned so a decision stays explicable after the source changes.

And the catalog is inverted: today programs are reached *through* a provider, which makes a well-fitting program invisible if nobody has been assigned to it — matching should end at a program, with providers resolved afterwards as the delivery layer.

## View and build the deck

Read the documents on GitHub, or open the rendered deck at **<https://hatchup-io.github.io/program-matcher/>**.

To render it locally (needs Node for `npx`):

```bash
npx @marp-team/marp-cli@latest deck/program-matcher.md --html --allow-local-files -o site/index.html
npx @marp-team/marp-cli@latest deck/program-matcher.md --pdf --allow-local-files -o site/program-matcher.pdf
```

Or preview live in VS Code with the Marp extension.

## Test

There are no tests. Lint the Markdown with the repository's settings in `.markdownlint.json`, for example:

```bash
npx markdownlint-cli2 "**/*.md"
```

CI does not run the linter; it only renders the deck.

## Deploy

`.github/workflows/deck.yml` (workflow "Deck") runs on the self-hosted runners. It renders `deck/program-matcher.md` with the `marpteam/marp-cli` Docker image into `site/index.html` and `site/program-matcher.pdf`.

- **Pull request** (from this repository only, never from a fork): builds the deck and leaves PDF + HTML on the run as the `program-matcher-deck` artifact, so a slide change is reviewable before it reaches the URL.
- **Push to `main`** (or a manual run): builds the deck and publishes `site/` to GitHub Pages at <https://hatchup-io.github.io/program-matcher/>, with `program-matcher.pdf` beside the HTML.
- **Published release**: attaches `program-matcher.pdf` to the release.

No swarm stack runs anything from this repository.

### Conventions

Markdown prose is one paragraph per line, never hard-wrapped — `.markdownlint.json` disables `MD013` accordingly.

## Related repositories

- [pathwise_backend](https://github.com/hatchup-io/pathwise_backend): the Pathinnova backend, which has the `funnels` and `programs` apps.
- [pathinnova-programmes](https://github.com/hatchup-io/pathinnova-programmes): researched programme requirements, imported into that backend.
