# ② Data storage — three tiers, one rule

> Answers: *نحوه‌ی ذخیره‌ی داده*. Sibling docs: [collection](01-data-collection.md) · [update](03-data-update.md) · [questions](04-client-questions.md) · [delivery](05-provider-delivery.md) · [action plan](06-action-plan.md)

## 2.1 The rule

> **Engines read Tier 1. Language reads Tier 2. Both must resolve to Tier 3.**

Every failure this design is trying to prevent is a tier violation. A verdict computed from prose is a wrong verdict that reads convincingly. A sentence quoted from a structured field is a sentence with no source. A field with no provenance is a number nobody can defend.

## 2.2 Tier 1 — structured

The typed, queryable facts an engine computes on: the program, its plans, its eligibility parameters, its objective ratings. This is the **only** tier eligibility and qualification read, and the only tier a verdict may depend on.

It already exists and mostly does not change. What changes is that each field belongs to a revision and carries provenance.

```python
class MigrationProgram(TimeStampedModel):
    slug, name, country_code, category, is_active
    current_revision = FK("ProgramRevision", null=True, related_name="+")

class ProgramPlan(TimeStampedModel):
    program, slug, name, amount, currency, covers_family_up_to, is_addon, is_active
```

Eligibility parameters — ban lists, enhanced-due-diligence lists, age minimums, dependent age limits, contribution floors, required attestations — stay data on the revision, read by the shared per-category engine. Values differ per program; the rule shape is code.

## 2.3 Tier 2 — narrative

The source documents, chunked by section, embedded, and searched. This is the only tier the chat may quote from, and nothing here is ever read by an engine.

```python
class ProgramChunk(TimeStampedModel):
    revision      = FK(ProgramRevision, related_name="chunks")
    section_type  = CharField(choices=SECTION_TYPES)
    ordinal       = PositiveSmallIntegerField()      # order within the document
    language      = CharField(max_length=8)
    content       = TextField()
    content_hash  = CharField(max_length=64, db_index=True)
    embedding     = VectorField(dimensions=3072)

    class Meta:
        indexes = [
            HnswIndex(name="chunk_embedding_hnsw", fields=["embedding"],
                      m=16, ef_construction=64, opclasses=["vector_cosine_ops"]),
            models.Index(fields=["revision", "section_type"]),
        ]
```

### Section taxonomy

The chunker splits on the sections these documents already have, because a section is the unit an advisor cites:

`overview` · `qualifying_routes` · `costs_and_fees` · `eligibility_rules` · `restricted_nationalities` · `physical_presence` · `family_inclusion` · `processing` · `permanent_residence` · `citizenship_pathway` · `benefits` · `differentiators` · `risks_and_caveats`

**Why not a fixed window.** A window chunker splits a route's amount from its conditions, and that pair — separated — is precisely how a confident wrong answer is produced: the amount retrieves cleanly, the condition that qualified it does not.

## 2.4 Tier 3 — provenance

```python
class ProgramSource(TimeStampedModel):
    publisher, document_date, language
    kind          = CharField(choices=["official", "expert", "provider", "legacy_seed"])
    uri           = CharField()        # URL or repository path
    content_hash  = CharField(db_index=True)

class ProgramRevision(TimeStampedModel):
    program, source
    status          = CharField(choices=["draft", "in_review", "approved",
                                         "published", "superseded"])
    effective_from  = DateField(null=True)
    effective_until = DateField(null=True)
    facts           = JSONField()      # the approved Tier-1 payload
    approved_fields = JSONField()      # field -> {value, approver, source_span, reason}
```

`approved_fields` is what makes a number defensible: it holds, per field, the approved value, who approved it, the span it came from, and the reason if a reviewer overrode the proposal.

## 2.5 The query

Hard constraints are SQL predicates on the same tables as the vectors, so they **pre-filter** rather than post-filter a top-k that was already wrong. A lateral join caps how many chunks any one program can contribute, so a long document cannot outvote a terse one.

```sql
WITH eligible AS (
  SELECT p.id, p.slug, r.id AS revision_id
  FROM migration_programs p
  JOIN program_revisions r ON r.id = p.current_revision_id
  WHERE p.is_active
    AND p.category = ANY(%(categories)s)
    AND NOT (r.facts -> 'eligibility' -> 'banned_nationalities' ? %(nationality)s)
    AND (r.facts -> 'eligibility' ->> 'minimum_investment')::numeric <= %(budget_ceiling)s
)
SELECT e.slug, AVG(c.score) AS program_score
FROM eligible e
CROSS JOIN LATERAL (
  SELECT 1 - (ch.embedding <=> %(query_vec)s::vector) AS score
  FROM program_chunks ch
  WHERE ch.revision_id = e.revision_id
  ORDER BY ch.embedding <=> %(query_vec)s::vector
  LIMIT 3                                    -- per-program cap
) c
GROUP BY e.slug
ORDER BY program_score DESC;
```

Two properties worth noting. A program excluded by the `WHERE` clause is excluded **for a stated reason**, which the loop records — see [questions §4.3](04-client-questions.md). And the whole thing is one query against one database, so there is no window in which the index and the catalog disagree.

## 2.6 Sizing

| | Today | At 200 programs |
|---|---|---|
| Programs | 17 | 200 |
| Chunks (~12 sections each) | ~200 | ~2,400 |
| Embedding storage (3072 dims × 4 bytes) | ~2.5 MB | ~30 MB |
| Full re-embed cost | cents | a few dollars |
| HNSW build | seconds | seconds |

This is a small index by any measure. The operational question is not capacity, it is **truth** — see [update](03-data-update.md).

## 2.7 Why pgvector rather than a dedicated vector database

**One transaction.** A revision and its chunks commit together. No dual-write, no reconciliation job, and no window where the index still offers a route the catalog retired.

**One query.** Filters and vectors in the same statement, so hard constraints pre-filter. A separate store forces the wrong shape: fetch top-k, then discard whatever the constraints eliminate, and hope enough survived.

**Nothing new to operate.** No extra service in the deployment, no extra backup rotation, no additional way for a deploy to fail. The database is already backed up nightly; embeddings ride along, and they are re-derivable from Tier 3 in any case.

Revisit at roughly 10⁵ chunks, or as soon as a second service needs the same index.

## 2.8 Honest sizing of the idea

At today's catalog size, vector search is **not** what makes the conversation converge — the [question loop](04-client-questions.md) is, and it would work over a plain queryset.

What retrieval buys is **grounding**, so every claim the chat makes carries a source, and **reach**, so the same design still works at two hundred provider-authored programs. Both are worth building. Confusing which one solves which problem is how this ends up as a vector store bolted onto a conversation that still wanders.

## 2.9 Failure modes

| Failure | Prevention |
|---|---|
| A verdict computed from prose | Engines are wired to Tier 1 only; no Tier-2 read exists in the eligibility path |
| A claim with no source | Tier-2 chunks always carry `revision_id`; the guardrail rejects an unsourced program claim |
| Index disagrees with catalog | Same transaction, plus the nightly drift scan |
| A long document dominating retrieval | Per-program cap in the lateral join; verified in shadow |
| Embeddings lost | Re-derivable from Tier 3 by `reindex_programs --force` |

## 2.10 Acceptance criteria

The eligibility path contains no read of Tier 2. Every chunk resolves to a revision and a source. The retrieval query pre-filters, demonstrated by a test where a banned nationality removes a program that would otherwise rank first. A full rebuild from Tier 3 reproduces the index byte-for-byte in content hashes.
