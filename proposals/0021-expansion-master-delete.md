# An expansion's generated rows do not outlive their master

- **Status:** draft
- **Issue:** <!-- link the discussion issue, if any -->
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/issues/6821

## The problem

`expansions` says one thing about ownership and means it: the generated child set belongs to the
expansion, and a span change replaces it wholesale. Every event on the master is accounted for by
that rule — except the last one.

The construct binds the master's **create** and **update**. It says nothing about the master's
**delete**, so the rows it generated are the one part of the record that survives it. A leave request
whose day rows were expanded is deleted; the day rows remain, each pointing at a request id that no
longer resolves. They are not visibly wrong anywhere — they are wrong everywhere that counts them: a
roll-up of days taken, a monthly report, a remaining-balance figure. The record is gone from the list
and its cost is still on the books.

Nothing else in the model closes the gap:

- **Referential integrity is not a rescue.** Where it is enforced the delete is *refused* instead —
  which is not better, only louder: the author is told the master cannot be deleted and given no way
  to remove rows they never entered. Where it is not enforced, the orphans are silent. The construct
  has no delete story in either case, and which of the two an author meets is a property of the
  deployment, not of the intent.
- **A composition does not imply it either.** An expansion child is commonly a composed child, but
  composition governs how a child is *reached and edited*, not what happens to generated rows when
  the thing that generated them ceases to exist.

The gap is the same shape as the one `manyToMany` had: a construct whose declared contract is wider
than the events it actually binds, with no diagnostic pointing at the difference.

## The proposed shape

None. This is a semantic completion of an existing construct, not a new key — there is nothing for
an author to write, and nothing to opt into. An intent that already declares an expansion gains the
behaviour by re-generating.

```yaml
expansions:
  - name: leave-days
    from: LeaveRequest        # deleting a LeaveRequest now removes the LeaveDay rows it generated
    into: LeaveDay
    between: { start: from, end: to }
    map: { day: period }
```

## Expected behaviour

Deleting an expansion's master removes the rows that expansion generated for it, and only those: the
rows selected by the same back-reference the (re)generation uses.

The removal is a **per-row delete through the child's own layer**, not a bulk statement — the child's
delete event has to fire for each row, so that whatever reacts to a hand-deleted row (a roll-up on
the child, a capacity guard, a downstream aggregate) reacts here identically. A cleanup that
short-circuits the child's layer would leave exactly the stale totals it exists to prevent.

Idempotent, like the rest of the construct: a redelivered master-delete finds no rows and does
nothing.

## Edge rules

- The cleanup is scoped by the **back-reference**, so a child row pointing at a different master is
  untouched, and an expansion never reaches beyond the set it owns.
- No write-back to the master: the master is gone, and the row count it may have carried (`count`)
  goes with it.
- The **ownership rule already stated** is what licenses this: an expanded child must not hold
  hand-entered rows. An author who needs rows to survive their master is describing a child the
  expansion does not own, and must not expand into it.
- Nothing changes for a master that is *updated* into an empty span — that is a span change, already
  a replacement, and already produces no rows.

## Prior art / workarounds

Today the author has to write the missing handler by hand: subscribe to the master's delete, select
the children by the back-reference, delete each one — a re-derivation of the expansion's own
ownership rule, kept in a second place, and easy to forget precisely because the construct reads as
if it were already handled. The alternative reached for instead is a cascade in the data store, which
buys the cleanup at the cost of a rule the model does not state and cannot see: it fires for rows the
expansion does not own, it applies to hand-entered children too, and it bypasses the child's layer,
so the events the totals depend on never fire.

## Specification text

**Anchor:** Declarative glue > expansions — child rows from a date span

A span change replaces the generated child set — never mix hand-entered rows into an expanded child.

> **Normative.** Deleting the master removes the rows the expansion generated for it. Only rows
> selected by the expansion's own back-reference are removed, and each is removed through the child's
> layer, so the child's delete event fires for every row and anything reacting to a deleted child
> reacts exactly as it would for a hand-deleted one. The removal is idempotent.

## DSL index

Omitted — this proposal adds no construct.
