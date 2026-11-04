# A filtered roll-up follows a status move of its child

- **Status:** released in [1.12](../versions/1.12.md)
- **Issue:** https://github.com/IntentFile/intent-specification/issues/163, https://github.com/eclipse-dirigible/dirigible/issues/7942
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8003

## The problem

0066 lets a roll-up reduce only the children that satisfy its `where:` terms, and 1.11 requires it
to "recompute the parent when a child enters or leaves the set as it does on a create or a delete".
The usual filter is on the child's status - a cancelled call-off stops consuming its contract - and
the usual way a status moves is not an edit of the record: a `transitions` button, a create-from's
`sourceStatus` flip and a `setRelationField` step each write the status by a targeted write, which
announces a transition and not an update. A roll-up that recomputed on the child's create, update,
delete and re-parenting only went on counting a cancelled call-off, or went on ignoring a reissued
one, until an unrelated edit of a sibling happened to fire.

This is a clarification of which writes "enter or leave the set", not a new key.

## The proposed shape

No new key. The example that must hold:

```yaml
transitions:
  - { name: CancelCallOff,  forEntity: CallOff, from: [ISSUED],    setStatus: CANCELLED }
  - { name: ReissueCallOff, forEntity: CallOff, from: [CANCELLED], setStatus: ISSUED }
rollups:
  - { name: contractCommitted, entity: CallOff, via: Contract, field: committed, op: sum, of: total,
      where: [ { field: Status, op: ne, value: CANCELLED } ] }
```

A contract with call-offs of 100 and 50 reads 150; cancelling the 100 reads 50 at once, and
reissuing it reads 150 again.

## Expected behaviour

A roll-up declaring `where:` recomputes its parent on every status move of a child - a transition,
a create-from's `sourceStatus` flip, a `setRelationField` step - as it does on the child's create,
update, delete and re-parenting. The recompute is the same reduction over the same filtered rows.

## Edge rules

- A roll-up without `where:` is not concerned: a status move writes no column it reads, and it
  keeps exactly its existing recompute moments.
- The recompute stays a reduction of the children read back, not a delta, so a status move that
  does not change the set writes nothing.

## Prior art / workarounds

None short of a hand-written listener on the child's transitions. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8003.

## Specification text

**Anchor:** Declarative glue > rollups - denormalised parent totals, the `where:` / `guardAt:` /
`message:` normative block, appended.

> **Normative.**
> A roll-up declaring `where` MUST recompute its parent when a transition, a create-from's
> `sourceStatus` flip or a `setRelationField` step moves a child's status, as it does on the
> child's create, update, delete and re-parenting. A roll-up without `where` is not concerned: a
> status move writes no column it reads.

## DSL index

No new row; the `rollups[].where` row is unchanged.
