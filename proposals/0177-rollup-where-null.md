# A roll-up counts the children whose relation is empty - `value: null` in `where:`

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7984
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8886

## The problem

A roll-up's `where:` cannot select the children whose property has no value. A term needs a
`value`, and every comparison other than a test for emptiness leaves an empty property out, so no
`{ field, op, value }` term can say "has none".

The case is a ledger. A reversal of a posted entry is a second entry that names the one it reverses
(`Storno`), and the reversed entry names its reversal (`ReversedBy`). Both carry the same reference
to the source document - a payment, a depreciation - so the source's count of its posted entries
reads 2 after a reversal, while the ledger has netted the pair to zero. A rule that freezes a
document while it has posted entries then keeps it frozen for good. The count has to leave out the
reversal and the entry it reversed, and both are "the relation is empty" filters.

## The proposed shape

```yaml
rollups:
  - { name: postedEntries, entity: JournalEntry, via: CustomerPayment, field: postedEntries,
      where: [ { field: Status,     op: eq, value: POSTED },
               { field: Storno,     op: eq, value: null },
               { field: ReversedBy, op: eq, value: null } ] }
```

## Expected behaviour

- **`null` tests for a value.** With `op: eq` the term holds exactly when the property has no value;
  with `op: ne`, exactly when it has one. It applies to a field and to a to-one relation alike, a
  status relation included ("no status yet").
- **Both readings of the filter agree.** The children reduced and a row the capacity check tests are
  selected by the same reading, as for every other term.
- **The count follows the reversal.** In the example, a posted entry counts 1; its reversal, which
  names it in `Storno`, is not counted; once the entry names the reversal in `ReversedBy`, the count
  is 0.
- **The same reading holds wherever these terms are taken**: a schedule's `where` and a
  create-from's `items: where`.

## Edge rules

- `null` is not the string `"null"`: a quoted `"null"` is a string literal.
- `null` on a status relation is not a status name and is not resolved against the seeded statuses.
- A roll-up's `where:` still takes only `eq` and `ne`.

## Prior art / workarounds

Hand-written event handlers that recompute the count from the store on every change of the child,
outside the file.

## Specification text

**Anchor:** Declarative glue > rollups, after the paragraph introducing `where:` / `guardAt:` /
`message:` and its first normative block.

A `where:` term whose `value` is `null` tests whether the property has a value at all - the one test
that selects the children where it has none, since every other comparison leaves an empty property
out:

```yaml
      where: [ { field: Storno, op: eq, value: null } ]   # not a reversal
```

> **Normative.**
> A `where:` term with `value: null` MUST hold exactly when the property has no value under
> `op: eq`, and exactly when it has one under `op: ne`, on a field and on a to-one relation alike,
> and MUST NOT be reported as an authoring error. A roll-up MUST apply it the same way to the
> children it reduces and to a row its capacity check tests. `null` MUST NOT be read as a string
> literal or as a status name. A schedule's `where` and a create-from's `items: where` read a `null`
> value under `eq` / `ne` the same way.

## DSL index

| Construct | What it does |
| --- | --- |
| [`rollups[].where`](#rollups--denormalised-parent-totals) | which child rows a roll-up reduces - `{ field, op, value }` terms, `eq` / `ne`, their AND; `value: null` tests for an empty property |
