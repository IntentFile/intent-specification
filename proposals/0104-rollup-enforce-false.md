# A roll-up keeps its balance against a capacity it does not enforce - `enforce: false`

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/166, https://github.com/eclipse-dirigible/dirigible/issues/7904
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8475
- **Amends:** [0066 - a roll-up counts the rows it is told to, and its ceiling is enforced at a status](0066-rollup-where-and-guard-at.md)

## The problem

In 1.11 a roll-up's `balance` is defined as `capacity - sum`, so it needs `capacity`, and a
`capacity` always installs the overdraw check on the child: the cross-model text calls it "the
overdraw check they install", and `guardAt:` (0066) moves that check to a status of the child
without removing it. That is right for money that must not be over-allocated. It is wrong for a
project's budget against the hours actually booked on it: hours booked past the budget are
information, not an error. The author wants the remainder on the project's record, and no refusal.

Two related gaps meet on the same roll-up. A parent with no capacity is read as capacity 0, so an
unbudgeted project shows a balance of `-sum`. And the balance follows only the sum: an edit of the
capacity on the parent leaves the balance stale until the next child event moves the sum.

## The proposed shape

```yaml
rollups:
  - { name: projectActualHours, entity: ProjectTimesheet, via: Project, field: actualHours,
      op: sum, of: approvedHours,
      capacity: budgetHours, balance: remainingHours,
      enforce: false }                 # measured against the capacity, never guarded by it
```

## Expected behaviour

- **No capacity check.** With `enforce: false` no capacity check is installed on the child, gated
  or ungated: a child that takes the sum past the capacity is stored. The sum and the balance are
  maintained as for any capacity roll-up, so the balance goes negative.
- **An empty capacity gives an empty balance.** For such a roll-up a parent whose capacity has no
  value has no balance, not `-sum`.
- **The balance follows the capacity.** An edit of the capacity on the parent's own create or
  update recomputes the balance in the same write, not only a child event.
- **The default is unchanged.** A roll-up without `enforce:`, or with `enforce: true`, still guards
  its capacity, and still reads an empty capacity as 0, which its `status:` thresholds depend on.

## Edge rules

- `enforce:` without `capacity:` is refused: there is nothing to enforce or not.
- `enforce: false` beside `guardAt:` or `message:` is refused: both word the check it removes.
- `enforce: false` on a roll-up whose parent is owned by another model is refused: the parent's
  write path belongs to its owner, so the balance could not follow an edit of its capacity. A
  child owned by another model with a local parent is accepted.
- A check that a total fits the parent's column, where one is installed, is not a capacity check
  and stays.
- Where the child is owned by another model, the report that "the capacity check is not
  installed" is not made for an unenforced roll-up: no check was asked for.
- A targeted write of the capacity by a workflow step does not recompute the balance; the next
  child event that moves the sum does.

## Prior art / workarounds

A plain `sum` roll-up into an `actualHours` field and a `calculatedOnUpdate` remainder on the
parent, which follows only the parent's own edits and not the children's. Reference
implementation: https://github.com/eclipse-dirigible/dirigible/pull/8475.

## Specification text

**Anchor:** Declarative glue > rollups - denormalised parent totals, after the `where:` /
`guardAt:` / `message:` normative block.

A capacity may be a measure rather than a limit. With `enforce: false` the roll-up keeps its sum
and its `balance` against the `capacity` and refuses nothing: a budget of hours against the hours
booked, where the overrun is what the record is there to show.

```yaml
rollups:
  - { name: projectActualHours, entity: ProjectTimesheet, via: Project, field: actualHours,
      op: sum, of: approvedHours, capacity: budgetHours, balance: remainingHours,
      enforce: false }
```

> **Normative.**
> A roll-up declaring `enforce: false` MUST NOT install a capacity check on the child, whether or
> not it would be gated, and MUST maintain `field` and `balance` as a capacity roll-up does. For
> such a roll-up a parent whose capacity has no value MUST have no balance, and the balance MUST be
> recomputed from the stored sum on the parent's own create and update, so that an edit of the
> capacity is reflected in the same write. A roll-up without `enforce:`, or declaring
> `enforce: true`, MUST behave as before. `enforce:` without `capacity:`, `enforce: false` together
> with `guardAt:` or `message:`, and `enforce: false` on a roll-up whose parent is owned by another
> model MUST be reported as authoring errors. A check that a total fits the parent's column, where
> one is installed, is not a capacity check and is unaffected. Where the child is owned by another
> model, the report that the capacity check is not installed MUST NOT be made for a roll-up
> declaring `enforce: false`.

## DSL index

| Construct | What it does |
| --- | --- |
| [`rollups[].enforce`](#rollups--denormalised-parent-totals) | `false` keeps the sum and the balance against the `capacity` without refusing a child that exceeds it |
