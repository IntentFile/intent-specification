# A retired target stops blocking its source

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/6814

## Why

An event-driven create-from is **at-most-once**: the generated creation looks for a target that already
back-references the source and returns that one instead of minting a second document. The guard asks a
question about **existence** — "is there a target for this source?" — and a voided document answers yes
forever. It keeps existing, and it keeps back-referencing the source.

So the source's one shot is spent at the first creation and nothing that later happens to the target
releases it:

1. An event mints invoice `INV-001` from timesheet 12.
2. `INV-001` is wrong; the void transition moves it to VOIDED. It still exists, still references
   timesheet 12.
3. The timesheet re-qualifies, or someone clicks the shared button: the guard finds the voided
   `INV-001` and hands it back. No replacement is ever created.

"Void and reissue" is an ordinary business flow — a document is retired **while keeping its number**,
and a fresh one is raised. Today it is inexpressible for an event-driven create-from, and the failure is
silent: the caller gets a 200 and the retired document.

An append cardinality is not the answer and reads like one. It removes the guard altogether, so the
author who reaches for it to get their replacement also gets a new document on **every** subsequent
qualifying event. The two are different questions: how many targets a source may have, and which
existing target still counts.

## What this adds

Nothing on the create-from. What a status **means** is already declared once, where the nomenclature is
seeded — the `stage:` classification a report's `scope:` resolves through:

```yaml
seeds:
  - name: invoice-statuses
    entity: InvoiceStatus
    rows:
      - { id: 1, name: DRAFT,     stage: draft }
      - { id: 3, name: ISSUED,    stage: live }
      - { id: 8, name: CANCELLED, stage: cancelled }   # a target in either of these
      - { id: 9, name: VOIDED,    stage: void }        # no longer blocks its source
```

The at-most-once guard reads that classification: a target whose status is classified `cancelled` or
`void` is **retired** and does not satisfy the guard, so the next qualifying event — or a click — mints
a replacement. A `draft` or `live` target still satisfies it.

Reusing the classification rather than adding a second key on the create-from is the point: two
vocabularies for "this row no longer counts" could only drift, and an author who has already said which
statuses retire a document should not have to say it again per rule.

## Expected behaviour

> **Normative.**
> The at-most-once guard of an event-driven create-from MUST be satisfied only by an existing target
> that is **not retired**. A target is retired when its `function: EntityStatus` value is a seed row
> classified `cancelled` or `void`.
> A retired target MUST be left as it is — superseding creates a new record and never edits, deletes or
> re-points the retired one, both of which remain readable.
> When the target carries no `function: EntityStatus` relation, or its nomenclature carries no `stage:`
> classification, the guard MUST remain satisfied by existence alone — the behaviour of a file that
> predates this rule is unchanged.
> A generator MUST report a warning when an event-driven create-from's target carries a lifecycle whose
> nomenclature is unclassified: that is the case where the guard reads as state-aware and is not.
> A file that adopts no `stage:` classification MUST regenerate byte-identical output.

## Edge rules

- **Redelivery idempotence is unchanged.** A redelivered event finds the live target it created and
  returns it; only a retired one is stepped over.
- **Several retired targets** may accumulate over repeated void-and-reissue cycles. The guard is
  satisfied by the non-retired one, of which there is at most one at any time.
- **A cross-model target** is seeded in its owner model, so its classification is not resolvable at the
  consumer; the guard stays existence-only there, the same limit `scope:` has.
- **`draft` does not retire.** A draft target is a document in progress, not a withdrawn one — treating
  it as retired would mint a second document while the first is still being written.
- **Cardinality is a separate axis.** An appending rule keeps no guard at all, so nothing about it
  changes; this rule is about what "once" means.

## Prior art / workarounds

- The `stage:` classification and a report's `scope:` — the same question ("which rows count?") asked of
  an aggregate, answered from the same declaration.
- A posting's reversal discriminates the original from its storno by a **link column** rather than by
  status. That works because a reversal is itself a document; a voided document has no counterpart to
  link, only a status.
- Today's workaround is to clear the target's back-reference by hand — which discards the only record of
  which source produced the voided document.
