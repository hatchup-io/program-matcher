# ⑤ Provider delivery — the program is matched, now who delivers it?

> Answers: *فلوی نمایش پروایدرها*. Sibling docs: [collection](01-data-collection.md) · [storage](02-data-storage.md) · [update](03-data-update.md) · [questions](04-client-questions.md) · [action plan](06-action-plan.md)

## 5.1 The inversion

Today the catalog is reached **through** a provider:

```
funnel ──▶ exclusive provider ──▶ their currently-effective assignments ──▶ programs shown
```

The catalog service returns the programs assigned to the funnel's exclusive provider. Matching therefore happens inside one provider's inventory, and **a program nobody has been assigned to is invisible however well it fits**. That is not a bug in the implementation; it is what the data flow says to do.

Program-first reverses the order:

```
applicant ──▶ matched program(s) ──▶ providers who can deliver them ──▶ consultant ──▶ booking
```

A provider-scoped funnel becomes a **filter on delivery**, not a constraint on matching. This is a real change to the catalog service, not a rewording — and it is the cheapest item in the [action plan](06-action-plan.md), with no dependency on the vector work.

## 5.2 Resolution

```python
def delivery_options(program, funnel=None, today=None):
    """Providers who may deliver this program, in presentation order."""
    assignments = (
        ProgramAssignment.objects
        .filter(program=program, effective_from__lte=today)
        .filter(Q(effective_until__isnull=True) | Q(effective_until__gte=today))
        .select_related("provider")
    )
    exclusive = [a for a in assignments if a.is_exclusive]
    if exclusive:
        return exclusive[:1]              # exclusivity means there is no choice
    options = [a for a in assignments if a.provider.is_active]
    if funnel and funnel.booking_scope == FUNNEL_PROVIDER_ONLY:
        options = [a for a in options if a.provider_id == funnel.exclusive_provider_id]
    return order_providers(options)       # §5.4
```

The exclusivity shortcut is not a policy choice — the database already enforces at most one open exclusive assignment per program, so where one exists it *is* the answer.

## 5.3 The three cases

### One exclusive provider

No choice is presented, because there is none. The card names the provider and says the program is delivered exclusively by them.

It must not be presented as though a selection was made on the applicant's behalf. A shortlist of one, dressed as a recommendation, is the thing that erodes trust in the whole result.

### Several providers

Presented as a comparison, ordered by an **explicit, auditable rule** — see §5.4.

### No provider

The program is **still shown**, marked *not currently deliverable*, and routed to a consultant.

Hiding a matched program because nobody has been assigned to it is exactly the silent failure the provider-first catalog has by construction: the applicant never learns that the best fit for them exists, and the business never learns there is demand for a program it has no provider for. That second signal is worth having — **an unmet-demand counter per program is a commercial input, not just a log line.**

## 5.4 Ordering — the decision we need

For a non-exclusive program with several providers, the order must be stated rather than inherited from whatever the queryset returned.

| Rule | Argument for | Argument against |
|---|---|---|
| **Consultant rating** | Applicant-aligned, already collected | Thin data on new providers; slow to move |
| **Response time / capacity** | Predicts whether the applicant is actually served | Punishes a good provider having a busy week |
| **Commercial terms** | Aligns with revenue | Not applicant-aligned; must be disclosed if used |
| **Round-robin** | Fair between providers, no gaming | Ignores fit entirely |

**Recommendation:** rating first, capacity as a tie-break, with commercial terms excluded from ordering and handled in the commission layer instead. Whatever is chosen, the ordering inputs are recorded on the presented result so a given order can be explained months later.

This is a business decision. It is listed in the [action plan](06-action-plan.md) as a blocker on W5's final slice, not on its start.

## 5.5 The provider card

What the applicant sees per provider, and where each field comes from:

| Field | Source |
|---|---|
| Provider name, logo | Provider profile (public-tier asset) |
| Delivers this program since | `ProgramAssignment.effective_from` |
| Exclusivity | `ProgramAssignment.is_exclusive` |
| Service fee and what it includes | Provider-authored commercial terms — Tier 1, provider-approved |
| Typical response time | Derived from booking history, not self-reported |
| Languages | Consultant profiles under the provider |
| Rating | Existing consultant rating aggregate |

Two rules. **Nothing self-reported that can be measured** — response time comes from booking data. And **program facts never differ between provider cards**: the programme's thresholds and timelines come from the program revision, so two providers cannot advertise different processing times for the same route.

## 5.6 From provider to booking

```
provider chosen ──▶ that provider's consultants (verified, active)
                ──▶ availability / timeslot
                ──▶ booking created, stamped with:
                       program, plan, program_assignment, funnel
```

The booking model already carries program, plan and assignment. Two changes:

**Where attribution comes from.** Today it is derived from the funnel's exclusive provider and only fills in for a single-candidate card. Program-first, it is derived from the **matched program and the chosen assignment**, so a marketplace booking against a matched program is attributed correctly instead of leaving the fields null.

**What it points at.** The stamp includes the program *revision*, so the attribution records the version of the programme that was actually sold. See [update §3.6](03-data-update.md).

## 5.7 API surface

| Endpoint | Purpose |
|---|---|
| `GET /programs/{slug}/delivery/` | Providers who can deliver this program, ordered, with the ordering inputs |
| `GET /programs/{slug}/delivery/{provider_id}/consultants/` | That provider's bookable consultants in this funnel |
| `POST /bookings/` | Unchanged shape; attribution now derived from the matched program |
| `GET /catalog/programs/` | Program-first catalog browse — **no longer provider-scoped** |

The last one is the breaking change, and it is deliberate. The funnel filter moves from the catalog query to the delivery query.

## 5.8 What this changes for exclusive funnels

Nothing visible, and that is the point. A funnel whose booking scope is provider-only still shows only that provider's consultants, still books only them, and still attributes the same way.

What changes is *why*: it is a filter applied at delivery, rather than a wall that decided which programs could be matched. The same deployment behaves identically while the system underneath stops being provider-shaped.

## 5.9 Acceptance criteria

A program with no assignment is returned by matching and marked not deliverable. A program with an exclusive assignment returns exactly one provider. A provider-only funnel returns the same consultants and bookings as today, verified against the existing test suite. Booking attribution fills in for a matched program regardless of funnel exclusivity. The ordering inputs are recorded on every presented delivery result. The catalog browse endpoint is program-first and scoped by category, not by provider.

## 5.10 Open decisions

The ordering rule (§5.4) — needed before W5's final slice ships.

Does an applicant matched to a program with no provider get routed to a general consultant, or to a waitlist against that program? Recommendation: both — consultant now, waitlist counter for the commercial signal.
