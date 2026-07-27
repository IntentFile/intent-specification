# Keyed aggregates, guard checks, and event-driven row posting

- **Status:** draft
- **Issue:** <!-- link the discussion issue, if any -->
- **Proposed for:** version 1.1 (see `versions/1.1.md` in this pull request)

## The problem

Three recurring business rules cannot be expressed in 1.0, and each of them is currently
hand-written code in every application that needs it.

**1. A total keyed by more than one relation.** `rollups` denormalise a total onto the parent of a
composition: one key, the child's own parent relation. Real inventory needs the signed sum of stock
movements per **product and store**; receivables need open exposure per **customer**; leave needs the
remaining allowance per **employee and year**. None of these is a parent total - each is a value in
its own right that other records must be able to reference and that lists must be able to show. Today
the only options are a read-only report (not referenceable) or a hand-maintained table.

**2. A precondition over such a total.** "Do not let this issue drive stock negative." "Hold this
order if it breaks the customer's credit limit." "Reject this leave request if the allowance is
exhausted." All three compare a keyed total against a limit; all three are written by hand, and each
implementation invents its own way to avoid racing the total's own maintenance.

**3. Derived rows on a document event.** A goods receipt that reaches its posted status has to emit
one signed movement per line into the stock ledger; a payroll run emits its payslip lines. The shape
is always the same - iterate the items, map fields, write through the target, and do not double-post
if the event is delivered twice - and it is always hand-written, which is where double-posting bugs
come from.

## The proposed shape

```yaml
# 1. a total keyed by two relations, materialised into its own entity
aggregates:
  - name: onHand
    of: StockMovement
    op: sum
    sum: quantity
    by: [Product, Store]
    into: ProductAvailability
    field: onHand

# 2. a precondition over that total, with three outcomes
entities:
  - name: StockMovement
    checks:
      - kind: guard
        aggregate: onHand
        minimum: 0
        message: "Insufficient stock"
        enabledBy: BLOCK_NEGATIVE_STOCK   # optional configuration gate
  - name: SalesOrder
    checks:
      - kind: guard
        aggregate: openExposure
        minimum: 0
        outcome: task                     # accept, then mark for a human step
        marker: withinCredit
  - name: LeaveRequest
    checks:
      - kind: guard
        aggregate: remaining
        minimum: 0
        outcome: reject                   # accept, file it already rejected
        setStatus: 4

# 3. derived rows emitted on a document event
posts:
  - name: goodsReceiptLedger
    event: POSTED
    forEach: items
    into: StockMovement
    idempotentBy: GoodsReceipt
    set:
      Date:         Receipt.Date
      Store:        Receipt.Store
      Product:      item.Product
      Quantity:     item.Quantity
      Direction:    1
      GoodsReceipt: Receipt.Id
```

## Expected behaviour

**`aggregates`.** On every create, update and delete of a source row, the target row for that row's
key-tuple is upserted and the aggregate recomputed from all source rows sharing the tuple, so a
replayed event converges instead of accumulating. A row with any grouping key unset belongs to no
tuple and is ignored. The recompute persists only the aggregate column - it must not overwrite the
rest of the target row, or it would revert a concurrent edit to a neighbouring column.

**`checks: kind: guard`.** The total is recomputed from the guarded entity's own rows for the
incoming record's key-tuple, excluding the record being updated, and the incoming value is added.
The comparison is deliberately NOT a read of the materialised target: a guard must not be able to
race the aggregate's own maintenance. `outcome` then decides:

| `outcome` | Companion | A violating write |
| --- | --- | --- |
| `block` (default) | - | is rejected with the authored message; nothing is stored |
| `task` | `marker:` (boolean field) | is stored; the marker is set false (true whenever the guard holds) |
| `reject` | `setStatus:` (status seed) | is stored; the record's status relation is set to that value |

`outcome: task` stamps a flag - it does not create or route to a task. A workflow decision reads the
marker and routes the record; the guard is the part that computes. Stating this explicitly matters,
because a construct that claimed to route work would be making a promise the write path cannot keep.

**`posts`.** When a source record enters the named status (or on create), one target row per
`forEach` child is written through the target's ordinary write path, so the target's numbering,
validations and derived fields all still apply. `idempotentBy` names the target's back-reference to
the source: the generator writes it and also uses it to skip an event whose rows already exist.

## Edge rules

- Every `by` name must be a to-one relation of both `of` and `into`; `into` is keyed by the same
  relations as the source groups by.
- A roll-up's parent may be cross-model (its `via` relation names a `uses` model): the child stays
  local because it owns the event, and the parent field is validated against the owner's model at
  generation time. `capacity` / `balance` / `status` stay local-only - they read the parent's own
  limit and status values.
- A guard's `aggregate` must name an aggregate whose `of` is the entity carrying the check.
- `outcome: task` requires a boolean `marker` field; `outcome: reject` requires `setStatus` and a
  status relation on the entity. A companion attribute belonging to another outcome is an authoring
  error, not something to ignore - a silently-ignored companion produces a check that appears to
  guard and does nothing.
- `idempotentBy` must be a to-one relation on the target pointing back at the source.
- Aggregates and roll-ups alike are recompute-on-event: eventually consistent, not transactionally
  exact. A guard, being recomputed synchronously from the source rows, is exact at the moment of the
  write.
- An aggregate over a `sensitive` field is itself sensitive wherever its target entity is personally
  scoped - otherwise hiding the leaf value and publishing its total would be a distinction without a
  difference.
- Moving a source row between tuples (editing a grouping key) repairs BOTH tuples: the one it
  joined and the one it left. Since the previous keys are unrecoverable after the write, a
  conforming generator must observe them before it. A tuple whose last row leaves keeps a zero
  total rather than disappearing.

## Prior art / workarounds

All three are hand-written today in applications built on the format. In one production suite the
row-posting shape alone accounted for eight near-identical classes of roughly 600 lines (six document
types, each iterating items, mapping fields, and re-implementing the same back-reference check), and
the three preconditions were three separate implementations of "sum related rows, compare, act" -
each with its own answer to the race against the total's maintenance, and one of them silently
depending on a table another component maintained asynchronously.

Comparable formats offer parts of this: ORM-level aggregate columns cover the single-key parent case
(`rollups` already does), and workflow engines can compare a value and branch, but the comparison
input has to be computed somewhere first. What is missing is the declarative pairing: a keyed total
that is a first-class row, and a precondition over it whose outcome is authored rather than coded.
