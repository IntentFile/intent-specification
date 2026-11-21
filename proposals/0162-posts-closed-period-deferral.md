# A `posts` row into a closed period waits, and is booked when the period reopens

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8772
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8830

## The problem

A [posting](#postings--source-document-to-ledger) whose target is period-locked does not write into a
closed period: it is deferred, and reopening the period posts it. The flattened
[`posts`](#posts--derived-rows-on-an-event) construct writes into the same kind of target - stock
movements, ledger rows - but the text says nothing of it. Its rows go through the target's ordinary
write path, which refuses a row dated in a closed period, so a goods receipt posted on the 31st of a
month the warehouse closed that morning fails its event, and nothing books it once the month is
reopened.

## The proposed shape

No new key.

```yaml
- name: StockMovement
  immutableInPeriod: { period: StockPeriod, date: Date }

posts:
  - name: goodsReceiptLedger
    event: POSTED
    forEach: items
    into: StockMovement
    idempotentBy: GoodsReceipt
    set: { Date: Receipt.Date, Product: item.Product, Quantity: item.Quantity, GoodsReceipt: Receipt.Id }
```

## Expected behaviour

- Before writing, every row the entry derives is asked whether its date falls in a closed period of
  the target's register. If any one does, nothing of the event is written: the source stays
  unposted, the deferral is reported naming the source, and the event is consumed rather than
  failed.
- When the period is reopened, every source the entry selects, dated inside the reopened window
  and not yet back-referenced through `idempotentBy`, is posted again through the entry's own body,
  with its period check.
- A target without a period lock is unaffected.

## Edge rules

- One closed row holds back the whole event: a `posts` event is never half-written.
- The reopen sweep needs `idempotentBy` to tell a posted source from an unposted one; an entry
  without it is deferred but not swept.

## Prior art / workarounds

Re-firing the source's event by hand after the reopen.

## Specification text

**Anchor:** Declarative glue > `posts — derived rows on an event`, after its normative block.

A `posts` entry into a period-locked target is deferred as a posting is: a closed period holds the
event back, and reopening the period books it.

> **Normative.**
> Where the target of a `posts` entry declares `immutableInPeriod`, a conforming generator MUST, before
> writing, ask every row the event derives whether its date falls in a closed period. If any one does,
> it MUST write none of them, MUST report the deferral naming the source, and MUST NOT fail the event.
> When the period is reopened, by a transition or a plain edit of the register, every source the entry
> selects, dated inside the reopened window and not yet back-referenced through `idempotentBy`, MUST
> be posted again through the entry's own body, its period check included. An entry declaring no
> `idempotentBy` MUST be deferred and is not required to be re-posted. A register write that leaves
> the period closed MUST post nothing.

## DSL index

No new row; the `posts` row is unchanged.
