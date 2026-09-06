# A roll-up's status is relinquished, not only set

- **Status:** draft
- **Issue:** none filed; reported against the implementation
- **Implementation:** [eclipse-dirigible/dirigible#7016](https://github.com/eclipse-dirigible/dirigible/issues/7016)

## The problem

A `rollups:` entry with `capacity` and `status` moves the parent to `statusWhenFull` /
`statusWhenPartial` as the summed children arrive. The 1.5 text says what happens when the sum
grows and nothing about what happens when it shrinks - and an implementation that reads it
literally models "money arrives" and forgets "money leaves".

An invoice of 1000, confirmed, receives an allocation of 1000: paid 1000, balance 0, PAID. The
allocation is deleted: paid 0, balance 1000 - and **still PAID**. The invoice now claims to be
settled while owing its full amount, and because a settlement allocates only to payable statuses
(ISSUED, SENT, PARTIAL), the next payment will not land on it either. The same happens when the
allocation is amended to 0 or re-parented to another invoice: every path that takes the sum back
to zero leaves the status where the last positive sum put it.

## The proposed shape

No new key. The existing declaration:

```yaml
rollups:
  - { name: invoicePaid, entity: Allocation, via: SalesInvoice, field: paid,
      op: sum, of: amount, capacity: total, balance: balance,
      status: Status, statusWhenFull: 7, statusWhenPartial: 6 }
```

A `statusWhenEmpty: <id>` was considered and rejected. A declared return status is wrong for every
document that entered the roll-up's region from a different status than the declared one - an
invoice paid straight from ISSUED, never CONFIRMED, would come back as CONFIRMED, a status it
never held. The status to return to is the one the roll-up displaced, and only the roll-up knows
it, so the roll-up remembers it.

## Expected behaviour

- The **first** move into `statusWhenFull` or `statusWhenPartial` - a move from a status that is
  neither of the two - records the status the parent held until then.
- Moves between the two roll-up-owned statuses (PARTIAL to PAID and back) keep that record
  unchanged.
- When the sum returns to zero and the parent holds one of the two roll-up-owned statuses, the
  recorded status is restored and the record cleared.
- A parent that holds any other status when the sum returns to zero is left alone. A document
  voided or cancelled by hand while partially paid stays voided when its allocation goes; the
  roll-up relinquishes only what it set.
- The restore happens on every path that recomputes the sum - the child's create, update, delete
  and re-parenting - and is written together with the recomputed amounts, so the parent's
  consumers observe one change.

## Edge rules

- Where the memory lives is the implementation's concern. A conforming implementation keeps it
  on the parent, system-owned: never editable, never rendered on a generated surface, never
  overwritten by a user's write of the parent.
- A roll-up-owned status with **no** recorded predecessor - a parent set to PAID by hand, or one
  paid before the implementation kept the record - is left as it is and the situation reported.
  Guessing a status is worse than keeping a wrong one visibly.
- With a declared `lifecycle:` on the parent, the moves the roll-up makes back are transitions
  like the moves in, and the edges MUST be declared for them.
- `statusWhenEmpty` is not a key of the format and MUST be rejected as unknown.

## Prior art / workarounds

A hand-written listener on the child's delete event that resets the parent's status - one more
place that knows the status ids, and one that still has to guess where to return to. Or a
`transitions:` button that lets a person "reopen" the invoice by hand, which is the failure the
roll-up exists to remove.

## Specification text

**Anchor:** Declarative glue > rollups — denormalised parent totals, appended to the paragraph
that introduces `statusWhenFull` / `statusWhenPartial`.

A status the roll-up sets, it also relinquishes. The first move into `statusWhenFull` or
`statusWhenPartial` records the status the parent held until then; when the sum returns to zero
and the parent still holds one of those two statuses, the recorded status is restored and the
record cleared. A parent holding any other status at that moment is left unchanged.

> **Normative.** A conforming implementation MUST restore the displaced status on every path that
> takes the sum to zero - the child's create, update, delete and re-parenting - in the same write
> as the recomputed amounts. It MUST relinquish only `statusWhenFull` / `statusWhenPartial`, never
> a status set by another writer. It MUST keep the displaced status system-owned: not editable,
> not rendered on a generated surface, not overwritten by a user's write of the parent. A
> roll-up-owned status with no recorded predecessor MUST be left unchanged and reported, not
> replaced by a guess. The format has no `statusWhenEmpty`.
