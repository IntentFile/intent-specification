# A flow's status write while a roll-up holds the status is displaced

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/163, https://github.com/eclipse-dirigible/dirigible/issues/7869
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8415

## The problem

0030 lets a capacity roll-up move its parent into `statusWhenFull` / `statusWhenPartial`, record
the status it displaced, and restore that status when the sum returns to zero. 0054 makes the
status a flow's column when a `setRelationField` step writes it. The two meet on an invoice that is
paid before its approval step completes: the roll-up moves the invoice to PAID, recording DRAFT;
the approval step then writes APPROVED over PAID. The roll-up moves the status only when its sum
moves, so nothing puts PAID back, and the invoice reads APPROVED with nothing left to pay. Had the
payment been removed instead, the invoice would have returned to DRAFT, a status the flow had
already left.

This is a clarification of how the two constructs compose, not a new key.

## The proposed shape

No new key. The example that must hold:

```yaml
rollups:
  - { name: invoicePaid, entity: Allocation, via: SalesInvoice, field: paid,
      op: sum, of: amount, capacity: total, balance: balance,
      status: Status, statusWhenFull: PAID, statusWhenPartial: PARTIALLY_PAID }
processes:
  - name: approval
    trigger: { onCreate: SalesInvoice }
    steps:
      - { name: approve, kind: userTask, args: { assignee: approver, form: ApproveInvoice } }
      - { name: approved, kind: serviceTask, args: { setRelationField: Status, value: APPROVED } }
```

An invoice paid in full before `approve` completes stays PAID when the step writes APPROVED, and
the recorded displaced status becomes APPROVED. Deleting the payment restores APPROVED.

## Expected behaviour

While a record stands in a status a capacity roll-up owns, a `setRelationField` step that writes
that record's status writes the displaced status instead of the status. The record keeps the
roll-up's status; when the sum returns to zero, the roll-up restores the status the flow reached,
not the one the payment found. The step's write is announced as before.

## Edge rules

- The rule covers a step that writes the trigger record's status, a one-hop step that writes a
  related record's status, and a step that sets several fields at once.
- A `transitions` entry naming the owned status in `from` is not redirected: naming it is how an
  author voids or cancels a partly paid document, and a voided document stays voided when its
  allocation goes (0030).
- A create-from's `sourceStatus` hook is not redirected: its protection against generating from the
  same source twice reads the status itself.
- A step writing a record whose status no capacity roll-up owns generates exactly as before.

## Prior art / workarounds

None: the flow's write overwrote the roll-up's status. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8415.

## Specification text

**Anchor:** Declarative glue > rollups - denormalised parent totals, after the normative block on
relinquishing the status ("A conforming implementation MUST restore the displaced status ...").
Relates to Processes & forms > processes > A status the flow writes is the flow's column.

A flow and a roll-up may both write the same status. While the roll-up holds it, the flow's write
lands on the status the roll-up displaced, so the roll-up hands back the status the flow reached.

> **Normative.**
> While a record stands in a status a capacity roll-up owns (`statusWhenFull` /
> `statusWhenPartial`), a `setRelationField` step that writes that record's status MUST write the
> displaced status instead, so the roll-up restores the status the flow reached, not the one the
> payment found, when its sum returns to zero. A transition naming the owned status in `from` and a
> create-from's `sourceStatus` hook are not redirected.

## DSL index

No new row; the `rollups` and `processes` rows are unchanged.
