# ① Data collection — how a program enters the catalog

> Answers: *فلوی جمع‌آوری داده*. Sibling docs: [storage](02-data-storage.md) · [update](03-data-update.md) · [questions](04-client-questions.md) · [delivery](05-provider-delivery.md) · [action plan](06-action-plan.md)

## 1.1 Today

A developer reads a source document and hand-writes a seed YAML file. Seventeen programs across three categories, one author each, no review step, and no machine-readable record of which sentence in the source produced which number in the seed.

The seeds are careful — several carry comments recording exactly this kind of judgement, such as leaving a restricted-nationality list empty on purpose because *an unstated rule is not a rule*. That care lives in comments, which means it is advisory to the next human and invisible to the system.

Three things break as the catalog grows. Provider self-authoring has no gate to pass through. A changed source has no diff to review, only a rewritten file. And the question that matters most in a regulated advisory product — *where did this threshold come from?* — has no answer beyond asking whoever wrote the file.

## 1.2 Source taxonomy and trust

| Kind | Examples | Trusted for | Not trusted for |
|---|---|---|---|
| **Official** | The authority's programme brief, the law, the implementing regulation, an official fee schedule | Every legal fact: thresholds, eligibility rules, restricted nationalities, processing authority | Nothing — but it is often silent, and silence is not permission |
| **Expert** | In-house methodology, advisory briefings, consultant experience | Weighting, objective ratings, judgement about who a program suits, realistic timelines | Legal fact — an expert's recollection of a threshold never overrides the official figure |
| **Provider** | The provider's own one-pager, their fee sheet, their marketing | Commercial terms they control: their service fee, their turnaround, what they include | Legal fact, and any claim about the programme itself |

**Conflict rule.** For a legal fact, official beats expert beats provider, and a conflict is surfaced to the reviewer rather than resolved silently. For a commercial fact, the provider is authoritative because it is their own term. A conflict that cannot be resolved leaves the field *unknown* and blocks publish if the field is required.

## 1.3 The intake state machine

```
   registered ──▶ extracted ──▶ in_review ──▶ approved ──▶ published ──▶ superseded
       │              │             │            │                          ▲
       │              │             │            └──────────────────────────┘
       │              │             │                  (next revision)
       │              │             └──▶ rejected  (extraction unusable; re-run or author by hand)
       │              └──▶ failed    (unparseable document; stays registered)
       └──▶ archived                 (source withdrawn)
```

**registered** — the document exists with its provenance stamp: publisher, document date, language, kind (official / expert / provider), URI or repository path, and a content hash. No provenance, no registration.

**extracted** — a model has proposed structured fields and narrative sections. This is a *proposal*, and it is never a published state.

**in_review** — a human is approving field by field.

**approved** — every required field carries an approval. The revision is publishable.

**published** — the revision is current; chunks are built and embedded. See [update](03-data-update.md).

**superseded** — a later revision took over. The row stays; decision cards and sessions still point at it.

## 1.4 Extraction proposes, humans approve

Each proposed field carries the span it came from:

```json
{
  "field": "minimum_investment",
  "value": {"amount": 250000, "currency": "EUR"},
  "confidence": 0.94,
  "source_span": {
    "page": 2,
    "text": "A donation of EUR 250,000 to a qualifying cultural or heritage project."
  }
}
```

Approving is therefore reading one sentence, not re-reading the document. The reviewer can **approve**, **edit** (recording the reviewer, the new value and a reason), or mark **unknown** — which is a valid, approvable answer and the correct one whenever the source is silent.

Three rules the extractor is held to, each of which the current seeds already follow by hand:

**Silence is not permission.** A source that says nothing about restricted nationalities yields `unknown`, never "unrestricted". A rule that was never stated is not a rule, and inventing its absence is how a sanctioned applicant gets told they qualify.

**Numbers are never inferred.** A threshold that is not stated in the document is not derived from a neighbouring one, and a range is not narrowed to a point.

**Currency is never converted.** The catalog quotes EUR, GBP, AUD, CAD and USD, and converting them would put a number on an applicant's card that appears in no source document. The amount keeps the currency it was quoted in — the existing schema already carries a currency alongside the amount for exactly this reason.

## 1.5 The intake contract

A program cannot publish while a required field is unapproved. The required set is the contract, and it differs by category because the categories genuinely differ in what they grant.

| Field | All | CBI | RBI | Immigration |
|---|:--:|:--:|:--:|:--:|
| Category, country, display label | ● | | | |
| Outcome (what the applicant actually gets) | ● | | | |
| Authority and legal basis | ● | | | |
| Cost floor + currency | ● | | | |
| Processing time | ● | | | |
| Family inclusion rules and dependent age limits | ● | | | |
| Restricted nationalities (or an explicit *unknown*) | ● | | | |
| Objective ratings (the ranking dimensions) | ● | | | |
| Qualifying routes and their amounts | | ● | ● | |
| Physical presence requirement | | | ● | |
| Path to permanent residence / citizenship | | ● | ● | ● |
| Visa-free access count | | ● | | |
| Route type and sponsor requirements | | | | ● |

**Coverage report.** The panel shows, per program, which required fields are approved, which are `unknown`, and which are missing. A program with unknowns can still publish — an honest unknown is a legitimate state — but the shortlist card shows it, and the question loop treats an unknown gate as *unverified*, not as *passed*.

## 1.6 Language

Source documents arrive in more than one language, and the conversation runs in several. Three separate concerns, deliberately kept apart:

**Source language** is recorded on the source and never rewritten. The original text is what a reviewer verifies against.

**Structured values are canonical English tokens** regardless of source language — the same rule the chat already follows, where extracted signal values keep the schema's English vocabulary whatever language the conversation is in.

**Narrative chunks keep their source language**, with a translation stored alongside when one exists. Retrieval runs over both, so a Persian source is citable in an English conversation and the reverse.

## 1.7 A new category is a code change

A new *program* inside an existing category is pure data and passes through intake alone. A new *category* needs an eligibility template in code — the rule shape, not the values — and intake refuses to register a program in a category with no engine behind it, rather than accepting data nothing can evaluate.

This is the existing rule, made structural: category templates are code; programs within a template are data.

## 1.8 Migrating the existing catalog

The seventeen seeded programs are backfilled as revisions against a synthetic `legacy_seed` source, published, and flagged `review_due` immediately. Nothing about their behaviour changes on day one — the engines read the same values — but every one of them now has a revision to diff against and a review queue entry.

The order of review is by exposure: the programs the funnel actually shows first.

## 1.9 API surface

| Endpoint | Purpose |
|---|---|
| `POST /programs/sources/` | Register a document with its provenance stamp |
| `POST /programs/sources/{id}/extract/` | Run extraction, producing a draft revision |
| `GET /programs/{slug}/revisions/{id}/fields/` | The per-field proposal with source spans |
| `POST /programs/{slug}/revisions/{id}/fields/{field}/approve/` | Approve, edit or mark unknown |
| `POST /programs/{slug}/revisions/{id}/publish/` | Publish; blocked while required fields are unapproved |
| `GET /programs/coverage/` | Coverage report across the catalog |

All of it is real panel UI. Per the platform rule, an operator action that exists only in Django admin is not finished.

## 1.10 Acceptance criteria

A source cannot be registered without publisher, date, language and kind. Extraction never writes a published value. A revision with an unapproved required field cannot publish, and the API says which field. Every published Tier-1 field resolves to a source span or to an explicit reviewer edit with a reason. A category with no eligibility engine cannot accept a program. The seventeen existing programs are published as revisions with no behavioural change and appear in the review queue.

## 1.11 Open decisions

Who approves a field — an internal domain reviewer for everything, or the provider for their own programs with an internal spot-check on the legal fields? The recommendation is provider-authored, internally approved for anything legal, provider-approved for their own commercial terms.

How is extraction quality measured? The proposal is the **approval-edit rate**: the share of proposed fields a reviewer changes rather than accepts. It is the honest measure of whether extraction is helping or just moving the typing around.
