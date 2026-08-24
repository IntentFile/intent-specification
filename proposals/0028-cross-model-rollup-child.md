# A roll-up whose counted child is owned by another model

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/6930

## The problem

A `rollups:` entry requires the CHILD - the rows being aggregated - to be declared in the same model
as the parent whose field it maintains. That leaves a natural total inexpressible whenever the rows
belong to another module, which is exactly where an n:m pairing puts them.

Accounts receivable is the canonical case. The allocation entity (invoice, payment, amount) is a
link entity, and it lives with the document that owns one side of the pairing:

- The `sales-invoices` model owns `SalesInvoiceCustomerPayment` and can therefore declare the
  invoice-side total today: `paid` / `balance` / PARTIAL-PAID, all local.
- The payment side of the SAME rows cannot be declared at all. `CustomerPayment.allocated` (the sum
  of the payment's allocation amounts) and the `unapplied` figure derived from it belong to
  `customer-payments`, whose model may not mention a foreign child.

The consequence is not a missing convenience. "Is this payment fully applied?" and "how much credit
does this customer have on account?" are answered by a register report over the allocations instead
of by a stored, filterable, sortable number - so they cannot be a list column, a dashboard figure, a
schedule's `where` condition, or a report dimension.

## The proposed shape

Two keys on a `rollups:` entry, declared by the module that owns the PARENT:

```yaml
uses:
  - { model: sales-invoices }

entities:
  - name: CustomerPayment
    fields:
      - { name: id,        type: integer, primaryKey: true, generated: true }
      - { name: amount,    type: decimal }
      - { name: allocated, type: decimal, readOnly: true }
      - { name: unapplied, type: decimal, readOnly: true }

rollups:
  # CustomerPayment.allocated = the sum of the payment's allocation rows, which sales-invoices owns.
  # unapplied = amount - allocated, the operationally interesting figure.
  - { name: paymentAllocated, entity: SalesInvoiceCustomerPayment, model: sales-invoices,
      parent: CustomerPayment, via: CustomerPayment, field: allocated,
      op: sum, of: amount, capacity: amount, balance: unapplied }
```

- **`model:`** - the `uses:` alias of the model that owns `entity:`. The same key, with the same
  meaning, that a `schedules:` entry already takes for a cross-model source.
- **`parent:`** - the local entity whose `field:` the total lands on. It is authored rather than
  derived because a foreign child's relations are not in this document: nothing here can walk `via`
  to a target the way a local roll-up does.
- **`via:`** - a to-one relation of the FOREIGN child that points at `parent:`.

The mirror direction (a local child, a foreign parent) needs no new key: the child's `via` relation
carries its own `model:` alias.

## Expected behaviour

A conforming generator maintains `field:` on the local parent from the foreign rows, on the same
terms as a local roll-up: recomputed from stored rows on every child create, update and delete, so
the value self-heals and re-delivery converges; written targeted, so a concurrent write to another
column of the parent is not reverted; and written only when it actually changes, so a chain of
totals above it terminates.

The reads and the subscription are the OWNER's: the aggregate is computed over the rows as the owner
stores them, and the events consumed are the ones the owner's model publishes about them. The
dependency edge stays one-way - the consuming module already declares `uses:` the owner - and the
owner model is unchanged and unaware.

## Edge rules

- `model:` must be a declared `uses:` alias.
- `parent:` is required with `model:` and refused without it (a local roll-up's parent is the target
  of `via`, and a second statement of it could only drift).
- `parent:` must be a LOCAL entity. A total landing in a third model is that model's roll-up to
  declare; writing it from here would invert the dependency edge.
- `via:`, `of:` and `by:` name properties of the foreign child and are therefore validated against
  the owner's generated model, not the local document - the same design-time split every cross-model
  reference uses. A property the owner does not declare, and a `via` that references some entity
  other than `parent:`, are both refused rather than generated: keying the aggregate on foreign ids
  and looking the parents up by them would produce wrong totals silently.
- `capacity:` / `balance:` / `status:` are writes on the local parent and work as usual. The
  capacity GUARD does not: the check that refuses a child row overdrawing the parent belongs to the
  child's own write path, which the owner module generates. A conforming generator must report that
  the guard is not installed rather than let a `capacity` pass for an enforced limit.
- Re-parenting: a foreign row moved from one parent to another leaves the parent it left holding a
  contribution that no event of the row's own names. That side is repaired only when the owner model
  publishes a re-key notice for the relation (it does so for its own aggregates and roll-ups over
  the same relation); the parent the row moved TO is always correct, and deleting and re-creating
  the row is exact either way.
- A restricted (`visibleTo:` / `sensitive:`) foreign field does not propagate its restriction to the
  local total - the flag is not in this document. The restriction must be declared on the local
  field.

## Prior art / workarounds

- **A register report** over the allocations, which is what the case does today: correct, but not a
  stored number, so it cannot be filtered, sorted, totalled elsewhere or read by another construct.
- **A hand-written listener** in the payments module subscribing to the allocation topic - which is
  precisely the glue the roll-up construct exists to remove, and has to be re-derived whenever the
  allocation entity changes.
- **Moving the total into the owner model**, i.e. declaring the payment's field in
  `sales-invoices`. It puts a payment column in the invoicing module, which owns neither the entity
  nor its surfaces, and breaks down entirely when two modules allocate against the same payment.
- The same direction is already available to `schedules:` (a cross-model source entity) and to
  `generates` (`fromUses:`), so the shape of the extension is settled - this applies it to `rollups:`.
