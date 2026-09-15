# A master's delete and the composition children it owns

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7100,
  https://github.com/eclipse-dirigible/dirigible/issues/7143 (a refusal is decided before the first
  cascade)
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7111,
  https://github.com/eclipse-dirigible/dirigible/pull/7183

## The problem

`composition: true` on a to-one declares ownership: the child is a detail of its master, managed
under it, with a NOT NULL foreign key because it cannot exist without it. Nothing in the model says
what happens to those children when the master is deleted, and the observed answer is: nothing.
The child rows stay behind, pointing at a key that no longer exists.

An absence-of-lines record is not the failure. The failure is that those rows are still read. No
surface renders them - a detail is reached through its master's page, and the master is gone - while
every report, roll-up and aggregate over the child counts them exactly as before:

```
VacationDay { Request: manyToOne -> VacationRequest, composition: true, required: true }
rollups: consumed days per VacationEntitlement, over VacationDay

DELETE VacationRequest 2                  -> accepted
GET    VacationRequest 2                  -> not found
GET    VacationDay ?Request=2             -> the five rows are still there
GET    VacationEntitlement 1              -> consumed 2 / balance 0 / EXHAUSTED
```

The entitlement is charged for a request nobody can open. The same shape exists on every
header-items document: invoice lines, order lines, allocation rows.

A model cannot express either of the two sane outcomes today, and the one it silently gets - orphan
rows - is neither.

## The proposed shape

The decision belongs to the composition, and the composition is authored on the child:

```yaml
entities:
  - name: SalesOrder
    fields:
      - { name: id, type: integer, primaryKey: true, generated: true }
    relations:
      - { name: items, kind: oneToMany, to: SalesOrderItem }

  - name: SalesOrderItem
    fields:
      - { name: id,       type: integer, primaryKey: true, generated: true }
      - { name: quantity, type: decimal }
    relations:
      # deleted with the order (the default; the key may be omitted)
      - { name: order, kind: manyToOne, to: SalesOrder, composition: true, whenMasterDeleted: cascade }

  - name: SalesOrderCopy
    fields:
      - { name: id,   type: integer, primaryKey: true, generated: true }
      - { name: note, type: string, length: 200 }
    relations:
      # the order cannot be deleted while a copy of it exists
      - { name: order, kind: manyToOne, to: SalesOrder, composition: true, whenMasterDeleted: refuse }
```

## Expected behaviour

Deleting a master:

- **`cascade`** (the default) - every child of that master is deleted, and each child's deletion is
  a deletion in full: whatever a direct delete of that row would do happens here too. Its own
  composition children go with it, so a chain of any depth unwinds; the reactions bound to its
  deletion observe it, so aggregates over the child relinquish what they counted; anything the
  model records about a deletion is recorded. The master's own deletion and every child's are one
  atomic unit - all of them are durable, or none is and the master is still there.
- **`refuse`** - the deletion of the master is rejected while any child of that relation exists, and
  the message names both entities. Nothing is deleted. The master becomes deletable once the
  children have been removed deliberately.

Either way a row is never left pointing at a master that is gone. `whenMasterDeleted` decides which
of the two outcomes; it cannot ask for the third.

A master may own several compositions with different answers - lines that cascade beside copies
that refuse. Every `refuse` is then decided **before the first cascade begins**, so a deletion that
ends refused touches no child row at all. Atomicity alone is not enough here: a child's deletion has
observable side effects that are not rolled back with its row - a change trail records the attempt,
a reaction may have fired - and a trail that says the lines were deleted, next to lines that still
exist, is the defect this ordering removes.

## Edge rules

- Valid on a to-one that declares `composition: true`, and only on the entity's **first**
  composition - the one that is its composition parent. On any other relation it is an error: a
  plain association owns nothing, so there are no children whose fate the master's deletion decides.
- Values are `cascade` and `refuse`. Any other value is an error. The key omitted is `cascade`.
- The behaviour is not conditional on the key: `cascade` is what a conforming generator does for
  every composition, authored or not, because it is what composition means. The key exists to
  choose `refuse`.
- Both outcomes hold for **every** writer, not only for a delete arriving over a public interface -
  a reaction, a schedule and a cascade from a further master reach the same rows.
- A composition whose master lives in another model is out of scope, as a cross-model composition
  itself is: ownership does not cross a model boundary.

## Prior art / workarounds

Relational schemas express the same choice on the foreign key (`ON DELETE CASCADE` / `RESTRICT`) and
ORMs express it on the association (a cascade of `REMOVE`, an orphan-removal flag). Neither is
reachable from a model that describes documents rather than tables, and a database-level cascade
would not do here: it deletes rows without the deletions being observed, so every aggregate over
the child keeps counting them - the defect this proposal fixes, arrived at from the other side.

Without the construct, an author writes the sweep by hand: a reaction on the master's deletion that
loads the children and deletes them. It is written once per composition, it is forgotten far more
often than that, and being a reaction it runs after the master's deletion has committed - so a
failure leaves exactly the orphans it was meant to prevent.

## Specification text

**Anchor:** Relations & multi-model > relations > Relation attributes

> **Normative.** A `composition: true` to-one declares that the owning entity's rows are OWNED by
> the referenced record. Deleting the referenced record MUST NOT leave them behind. A conforming
> generator MUST, for every composition, either delete the children with the master or reject the
> master's deletion; the choice is `whenMasterDeleted` on the composition relation, and its default
> is `cascade`.

- **`whenMasterDeleted: cascade`** (the default; the key may be omitted) - deleting the master
  deletes every child of this relation first. Each child's deletion MUST be a deletion in full: its
  own composition children are deleted with it, the reactions bound to its deletion observe it (so
  aggregates and roll-ups over the child relinquish what they counted), and whatever the model
  records about a deleted row is recorded. The master's deletion and the children's MUST be one
  atomic unit: either all of them are durable, or none is and the master still exists.
- **`whenMasterDeleted: refuse`** - deleting the master MUST be rejected while any child of this
  relation exists, with a message naming the master and the child entity, and nothing MUST be
  deleted. The master becomes deletable once the children have been removed.

> **Normative.** Where a master owns several compositions, every `refuse` MUST be decided before the
> first cascade deletes a child row, so a refused deletion touches no child - including whatever a
> child's deletion would have recorded or triggered outside the transaction.

> **Normative.** Both outcomes MUST hold for every writer that can delete the master, not only for a
> deletion requested through a generated interface: a reaction, a scheduled write and a cascade from
> a further master reach the same rows.

`whenMasterDeleted` is valid only on a relation that declares `composition: true`, and only on the
entity's first composition (its composition parent). Elsewhere, and for any value other than
`cascade` or `refuse`, it MUST be rejected.

## DSL index

| Construct | What it does |
| --- | --- |
| [`relations` / `whenMasterDeleted`](#relation-attributes) | whether deleting a master deletes the composition children it owns, or is refused while they exist |
