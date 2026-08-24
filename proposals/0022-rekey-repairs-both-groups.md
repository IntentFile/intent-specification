# Re-keying repairs both groups, whatever wrote the key

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/6819
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/PENDING

## The problem

A derived total is attached to a **group** — the tuple of an `aggregates` key set, or the parent of a
`rollups` child. Moving a record between groups is an ordinary business act: a cost centre is corrected,
a task is dragged to another sprint, a line is moved to another shipment, a claim is reassigned to
another approver. The record is neither created nor destroyed; one column changes.

1.5 already says what must happen for `aggregates`: both sides are repaired, and "the previous keys
cannot be recovered after the write, so a conforming generator observes them before it". Two gaps sit
next to that sentence.

**Roll-ups are not held to it.** The `rollups` section says nothing about re-parenting at all, and the
companion proposal for the update recompute (0021) leaves the vacated parent as a MAY. So a `sum`
roll-up over a re-parented line is specified to correct the parent that received it and to go on
counting it in the parent it left — a total that is wrong on both ends of one edit, self-correcting only
when some unrelated child of the vacated parent happens to change. Nothing about the construct justifies
the difference: a roll-up is an aggregate over one key.

**The repair is attached to one writer, not to the write.** The observation "before the write" is
naturally implemented on the path that carries a whole record — a user's form submit. But a generator
that also emits *targeted* writers (a workflow step that sets a field, a lookup that resolves a
relation, a task form that persists the fields it edited) has second, third and fourth ways to move
exactly that column. Those writers deliberately do not raise the record's ordinary change event, so
neither side of the move is recomputed and the wrong totals never converge. The move that most deserves
the repair — the one a *process* makes, on records nobody re-opens by hand — is the one that silently
skips it.

## The proposed shape

No new syntax. Existing declarations, re-parented by an ordinary edit and by a process step:

```yaml
rollups:
  - { name: sprintPoints, entity: Task, via: Sprint, field: points, op: sum, of: estimate }

aggregates:
  - name: spendByCentre
    of: Expense
    op: sum
    sum: amount
    by: [CostCentre, Period]
    into: CentreSpend
    field: spend

processes:
  - name: reassign
    steps:
      - { name: move, setRelationField: { relation: CostCentre, value: 7 } }
```

Moving a `Task` to another `Sprint` must leave BOTH sprints' points right. Re-pointing an `Expense` at
another cost centre must leave BOTH centres' spend right — whether a person edited the record or the
`reassign` step wrote the relation.

## Expected behaviour

- When a write changes a **grouping column** — a key of any `aggregates` entry over the record, or the
  `via` relation of any `rollups` entry whose child it is — the totals of BOTH the group the record left
  and the group it joined MUST be recomputed.
- This MUST hold for **every** write the generator emits, not only the one that carries a whole record.
  A targeted / partial write that moves a grouping column MUST repair both sides even though it raises
  no ordinary change event for the record.
- Each repair is the same reduction the construct already specifies: read back the records that now
  belong to the group and reduce them. Neither side is a delta, so a re-delivery converges.
- The previous value of a grouping column is not recoverable after the write, so a conforming generator
  MUST observe it before writing.

## Edge rules

- The repair is conditional on a grouping column having **actually changed** — an edit that writes the
  same value MUST NOT recompute, so an unchanged record costs nothing and a cascade over a multi-level
  composition still terminates at rest.
- A record whose grouping column becomes unset leaves its group and joins none; the vacated group is
  still repaired. (`aggregates` already ignores a record with a key unset.)
- A group whose last contributing record leaves keeps its target row / parent field at the empty value
  (zero for a sum or a count, unset for `latest`); nothing is deleted.
- Repairing the vacated group MUST NOT re-raise the record's ordinary change event: the record moved
  once, and a second change event would re-fire every reaction, notification and integration bound to
  it. Whatever signal carries the repair is for the derived-total handlers alone.
- Only the derived columns are persisted, as for every other recompute, so a concurrent edit to another
  column of the group's row is never reverted.

## Prior art / workarounds

Modelling around it means declaring the grouping relation immutable and making the user delete and
re-create the record to move it — losing its identity, its history and anything referencing it, to work
around a missing repair. The alternative workaround, a periodic job that re-reduces every group, is a
reconciliation batch standing in for an event the writer already knew about.

## Specification text

**Anchor:** Declarative glue > rollups — denormalised parent totals

Add, after the paragraph on eventual consistency:

**Normative.** Moving a child to another parent — an edit of its `via` relation and nothing else — MUST
leave both parents' totals right: the parent that received the child recomputes, and so does the parent
it left, which MUST NOT go on counting a child that is no longer its own.

**Anchor:** Declarative glue > aggregates — keyed cross-entity totals

Replace the paragraph beginning "Changing a grouping key MOVES a source row between tuples":

Changing a grouping key MOVES a source row between tuples, and both sides are repaired: the tuple it
joined is recomputed, and so is the tuple it left, so no tuple keeps a contribution from a row that is
no longer in it. The previous keys cannot be recovered after the write, so a conforming generator
observes them before it. A tuple whose last contributing row leaves keeps its target row with a zero
total rather than disappearing.

**Anchor:** Declarative glue (section preamble)

Add:

**Normative.** The repair of a moved record belongs to the WRITE, not to one writer. Where a generator
emits several ways to write a record — a form submit that carries the whole record, and targeted writers
such as a process step's field setter, a lookup that resolves a relation, or a task form that persists
what it edited — every one of them MUST observe the grouping columns and repair both groups. A targeted
writer raises no ordinary change event by design; the repair MUST reach the derived-total handlers
without turning into one.
