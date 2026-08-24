# A roll-up recomputes on the child's update, whatever it aggregates

- **Status:** released in [1.6](../versions/1.6.md)
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/6820
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/PENDING

## The problem

1.5 states when a roll-up recomputes only for the counting case, and states it too narrowly: "A count
roll-up keeps a counter on a parent current on the child's create / delete."

Create and delete are not the only ways the set of children of a parent changes. A child changes
parents by an ordinary **edit** of its own parent relation — dragging a task to another sprint,
reassigning a loan to another member, moving a stock line to another shipment. No child is created and
none is destroyed, so on the create/delete reading nothing recomputes: the parent the child moved to
does not count it, and the parent it left goes on counting it. Both counters are wrong, and because a
roll-up is a recompute-on-event construct they stay wrong until some unrelated child of the same parent
happens to be created or deleted.

The counting case is exactly the one where this is invisible for a while: an off-by-one counter reads
as plausible, unlike a total that is visibly short by the amount of a line.

Read as written, the sentence also implies that the event set depends on what the roll-up aggregates,
which is not a property of the construct — the recompute is the same read of the parent's children
whatever it then reduces them to.

## The proposed shape

No new syntax. The same declaration, with the recompute stated once for every `op`:

```yaml
rollups:
  - { name: memberLoanCount, entity: Loan, via: member, field: loanCount }   # count
  - { name: sprintPoints, entity: Task, via: Sprint, field: points, op: sum, of: estimate }
```

Moving a `Loan` to another `Member` — an edit of `Loan.member` and nothing else — must leave
`Member.loanCount` correct on the member that received it.

## Expected behaviour

A roll-up recomputes the affected parent on the child's **create, update and delete**, for every `op`.
The recompute is the same one in all three cases: read the children that point at the parent, reduce
them, and persist the derived field only if the value moved.

On an update the affected parent is the one the child names **after** the edit — so an edit that
re-parents the child corrects the parent it moved *to*. Correcting the parent it moved *away from*
needs the child's previous parent, which the update event does not carry; a generator that publishes a
distinct event for a moved key MAY use it to recompute the vacated parent as well. Without that, the
vacated parent converges the next time one of its own children changes.

## Edge rules

- The update recompute is not conditional on the edit having touched a field the roll-up reads. The
  handler is idempotent — it reduces the children read back from the store rather than applying a
  delta — so recomputing after an unrelated edit writes nothing and is not observable.
- The write remains guarded on the value actually changing, so the cascade over a multi-level
  composition still terminates at rest.
- Only the derived columns are persisted, as for the create and delete recomputes, so a concurrent
  edit to another column of the parent is never reverted.

## Prior art / workarounds

`aggregates` — the keyed sibling of `rollups` — has been normative since 1.1 on exactly this point:
"On each create, update and delete of a source row … MUST upsert the target row for that row's
key-tuple and recompute", and it goes further with a re-keyed event that drops the total left behind on
the tuple a row moved out of. `rollups` describe the same recompute-on-event machinery over one key and
should not differ in when they run.

Modelling around it means declaring the parent relation immutable and making the user delete and
re-create the child to move it — which loses the child's identity, its history and anything referencing
it, to work around a missing event.

## Specification text

**Anchor:** Declarative glue > rollups — denormalised parent totals

Replace the opening sentence of the section body:

A count roll-up keeps a counter on a parent current as its children change. With `op: sum` the roll-up
keeps `field` equal to the sum of the children's `of` field, can maintain a `balance`
(= `capacity - sum`), and can flip a `status` relation to `statusWhenFull` / `statusWhenPartial`. Sum
roll-ups **compose transitively** across a multi-level composition (a leaf edit updates the mid total,
then the top total); recomputation stops when values stop changing.

And add, after the paragraph on eventual consistency:

**Normative.** A roll-up MUST recompute on the child's create, update and delete, whatever its `op`
reduces the children to — the event set is a property of the construct, not of the aggregation. On an
update the affected parent is the one the child names after the edit, so an edit that RE-PARENTS a
child MUST correct the parent it moved to; a generator that publishes a distinct event for a moved key
SHOULD also recompute the parent it moved away from, which otherwise converges on the next change to
one of that parent's own children. The recompute MUST be a reduction of the children read back for the
parent rather than an applied delta, so recomputing after an edit that touched nothing the roll-up
reads writes nothing.
