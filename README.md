# Bayat Group — Program Matcher

Design repository for a goal-directed chat that converges on a migration program, backed by a vector-indexed program catalog.

This repo holds the proposal, not the implementation. The system it describes is built in [`hatchup-io/bayatgroup-backend`](https://github.com/hatchup-io/bayatgroup-backend), under `apps/funnel_modules/bayat_group/` and `apps/programs/`. The design lives here so it can be reviewed and presented before any code is written.

## Contents

| Path | What it is |
|---|---|
| `deck/program-matcher.md` | The presentation (Marp). Internal engineering audience. |
| `docs/ARCHITECTURE.md` | The spec behind the deck — data model, loop, rollout, open questions. |

## The proposal in three sentences

The Bayat Group guided chat runs a fixed six-phase discovery plan and only ranks programs once the conversation is over, so it can take up to thirty-six turns to reach a shortlist that was decidable at turn eight.

We give the session an explicit candidate set, select each question by the information it would gain over that set, and stop as soon as no remaining unknown can reorder the result.

The program knowledge base — today static markdown that no runtime code reads — is chunked, embedded and stored in `pgvector` alongside the catalog, so the chat is anchored to real program facts from the second turn and every claim it makes carries a source.

## Building the deck

The deck is [Marp](https://marp.app) markdown. CI renders it to PDF and HTML on every push and attaches both to the workflow run; tagging a release attaches them to the release.

Locally:

```bash
npx @marp-team/marp-cli@latest deck/program-matcher.md --pdf --allow-local-files
npx @marp-team/marp-cli@latest deck/program-matcher.md --html --allow-local-files
```

Or preview live in VS Code with the Marp extension.

## Conventions

Markdown prose is one paragraph per line, never hard-wrapped — `.markdownlint.json` disables `MD013` accordingly.
