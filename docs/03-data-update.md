# ③ Data update — what happens when a program changes

> Answers: *نحوه اپدیت داده*. Sibling docs: [collection](01-data-collection.md) · [storage](02-data-storage.md) · [questions](04-client-questions.md) · [delivery](05-provider-delivery.md) · [action plan](06-action-plan.md)

## 3.1 Revisions, not edits

A published program is immutable. A change creates a new revision; publishing flips which revision is current, bounded by `effective_from` and `effective_until` — the same shape provider assignments already use.

Three things this buys that in-place edits cannot.

**A decision stays explicable.** A card produced in March cites the revision it was produced from. When the threshold moves in June, the March card still reads correctly instead of quietly becoming wrong — and a consultant reopening that file can see the programme as it was described to the applicant at the time.

**Change is reviewable.** A revision diff is what the reviewer approves, so a provider's edit to a cost floor is visible *as* a cost-floor change rather than as a rewritten file.

**Rollback is a pointer move**, not a restore from backup.

## 3.2 The publish algorithm

Publishing is one transaction. If any step fails, nothing moved.

```
BEGIN
  1. assert revision.status == "approved"
  2. assert every required field for the category is approved
  3. supersede: current_revision.status = "superseded",
                current_revision.effective_until = today
  4. revision.status = "published", revision.effective_from = today
  5. program.current_revision = revision
  6. chunk the revision's narrative sections
  7. for each chunk: if content_hash exists on the superseded revision,
                     copy its embedding; else enqueue for embedding
  8. write chunks
COMMIT
  9. embed the enqueued chunks (async), then mark the revision index_complete
```

Step 7 is the whole cost story: **only changed sections are re-embedded**. A revision that corrects one processing time re-embeds one chunk, not the document.

Between commit and step 9, the revision is current but its new chunks have no vectors. Retrieval treats a chunk without an embedding as absent and falls back to the superseded revision's chunk for that section, so the conversation degrades to *slightly stale prose* rather than to *nothing*. The window is seconds.

## 3.3 Triggers

| Trigger | Path |
|---|---|
| Revision published | The algorithm above |
| Source document replaced | New source → new extraction → new revision → review → publish |
| Seed file changed | Same path; a seed is a source of kind `legacy_seed` like any other |
| Manual repair | `manage.py reindex_programs [--program slug] [--force]` |
| Nightly | Consistency scan (§3.4) |

`--force` re-embeds regardless of hash, and exists for the case where the embedding model itself changes. A model change is a catalog-wide re-embed and a deliberate, announced operation — never a side effect of a deploy.

## 3.4 The drift scan

A nightly task, alongside the platform's existing periodic scans. Seven assertions:

1. Every active program has a current revision.
2. Every current revision has `status = "published"`.
3. Every published revision has at least one chunk per required section.
4. Every chunk's stored `content_hash` matches a hash of its content.
5. Every chunk has an embedding of the expected dimension.
6. No chunk belongs to a revision that is not current or superseded-but-referenced.
7. Every Tier-1 required field on a current revision has an entry in `approved_fields`.

A failure raises an operator notification and names the program. It does not self-heal: silent repair hides the fact that something wrote a state nothing should have written.

> **A chunk set that has silently drifted from the catalog is worse than an error.** An error is visible. Drift answers confidently, and wrongly.

## 3.5 Change mid-session

An applicant is eight turns into a conversation when the program they are converging on publishes a new revision.

**The session pins the revision it started on.** The conversation, the retrieval, the eligibility verdict and the shortlist all read the pinned revision through to the end. The next session gets the new one.

Without pinning, an applicant can be told two different things about the same programme inside one conversation, and the transcript will not explain why.

Three sub-cases:

**The pinned revision is superseded mid-session.** Nothing changes for that session. The card it produces records the pinned revision id, and the panel shows a *superseded during session* marker so a consultant reading it later knows why the numbers differ from today's.

**The pinned revision is rolled back** (published in error, withdrawn). This is the one case where a session must be interrupted: the applicant is told the programme information was withdrawn, the session is re-pinned to the now-current revision, and the affected signals are re-evaluated. Rare and loud, by design.

**A program is deactivated mid-session.** It leaves the candidate set with the exclusion reason `program_withdrawn`, which surfaces in the shortlist rather than the program silently vanishing.

## 3.6 Saved decision cards

A card stores the revision id of every program it names. A card is therefore reproducible: given the card, the exact prose and thresholds the applicant was shown can be reconstructed months later.

This matters commercially as well as legally — a booking's attribution points at a program *and* the revision that was sold.

## 3.7 Staleness

Every program carries a `review_due` date, set from its source kind: official sources age slowly, provider commercial terms age quickly.

An overdue program is **flagged, not dropped**. The panel shows it in a review queue, and the shortlist card states *verified as of ‹source date›*. Quietly shrinking the catalog because a source is old is a worse failure than showing a date and letting a consultant judge.

## 3.8 Cost

Re-embedding is cheap enough that it should never shape a decision here: a full catalog rebuild at today's size costs cents, and at two hundred programs, a few dollars. The expensive thing is a wrong number, not an embedding call.

## 3.9 Acceptance criteria

A published revision cannot be mutated. Publishing is atomic — a failure at any step leaves the previous revision current. Only changed chunks are re-embedded, demonstrated by a test that publishes a one-field change and asserts a single embedding call. A session pinned to a revision produces identical output before and after a subsequent publish. The drift scan detects, and reports without repairing, each of the seven failures above. A rolled-back revision interrupts its pinned sessions exactly once.

## 3.10 Open decisions

Review cadence per source kind — how stale is too stale before a program is flagged? Recommendation: 12 months for official, 6 for expert, 3 for provider commercial terms.

Does a withdrawn program stay visible in a shortlist already delivered to an applicant? Recommendation: yes, marked withdrawn, because an applicant who was shown something should be able to see what happened to it.
