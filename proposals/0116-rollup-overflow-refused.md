# A child whose parent total would not fit is refused and stores nothing

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/163, https://github.com/eclipse-dirigible/dirigible/issues/8096
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8427

## The problem

A parent's total is a column with a bound, and each child amount can fit its own column while the
total the parent would hold does not. 1.11 is silent on what happens then, on both of the shapes a
parent total takes:

- **Document totals.** A line of a `function: Document` master is stored, and the master's totals
  are then recomputed. When the recomputed total did not fit, the caller got a fault while the
  line was already stored, and the footer no longer matched the lines.
- **`rollups`.** A roll-up recomputes after the child is stored ("recompute-on-event"). Each child
  write was accepted because its own amount fit; the recompute then failed on the parent's column,
  and the parent's total stayed stale.

This is a clarification of what a conforming generator must refuse, not a new key.

## The proposed shape

No new key. The example that must hold, a document whose net total, the sum of its lines, is bounded
to six digits with two decimals:

```yaml
entities:
  - name: SalesOrder
    function: Document
    fields:
      - { name: net, type: decimal, precision: 6, scale: 2 }
  - name: SalesOrderItem
    function: DocumentItem
    fields:
      - { name: amount, type: decimal }
    relations: [ { name: order, kind: manyToOne, to: SalesOrder, composition: true } ]
```

A first line of 6000 is stored; a second line of 6000 is refused with a 400, there is still one
line, and the header's net is still 6000. A `sum` roll-up whose target has the same bound behaves
the same way.

## Expected behaviour

- A child write whose parent total would exceed the bound of the parent's column is refused as a
  validation outcome (400 where the surface is HTTP), never a fault, naming the parent, the field
  and the value it would become - for example "SalesOrder.net would become 12000.00, more than the
  9999.99 it can hold".
- Nothing is stored: not the child, not a partial total.
- A document item and its master's recalculated totals are one unit of work, so any failure of the
  master's write rolls the line back.
- A `sum` roll-up onto a local parent is checked on the child's write, before the child is stored:
  the parent's children are summed with the roll-up's `where` applied and the incoming row counted
  as written.

## Edge rules

- A parent owned by another model is not pre-checked: its column is bounded by the model that owns
  it. Its capacity check, where one is declared, is unchanged.
- The fit check is not a capacity check: it is installed whatever the roll-up declares about its
  capacity.
- A delete only lowers a sum and is not concerned.
- On the roll-up path only the fit is pre-checked; another refusal the parent's write may meet
  during the recompute is not.

## Prior art / workarounds

None: the overflow surfaced as a fault after the child was stored. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8427.

## Specification text

**Anchor:** Declarative glue > rollups - denormalised parent totals, after the normative block
"A roll-up MUST recompute on the child's create, update and delete ...".

A child amount that fits its own column can still take its parent's total past the bound of the
parent's column. The child is refused before anything is stored, whichever way the total is kept.

> **Normative.**
> A child write whose parent total would exceed the bound of the parent's column MUST be refused
> with a validation outcome (400 where the surface is HTTP) naming the parent, the field and the
> value it would become, and MUST store nothing: a document item and its master's recalculated
> totals are one unit of work, and a `sum` roll-up onto a local parent is pre-checked on the child's
> write, `where` applied and the incoming row counted. A cross-model parent is not pre-checked.

## DSL index

No new row; the `rollups` row is unchanged.
