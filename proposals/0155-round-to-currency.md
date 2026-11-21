# A money amount is rounded to its currency's minor units

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8307, https://github.com/eclipse-dirigible/dirigible/issues/8327
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8837

## The problem

A decimal is stored to its column's `scale`, whatever currency the amount is stated in. A currency
with no minor units (a yen, `rounding: 0`) or with three (a dinar, `rounding: 3`) is therefore
billed to two decimals: an invoice line of 3 x 333.33 in yen stores a net of 999.99, a fraction of
a yen nobody can pay, so the receivable never reaches zero through a real payment.

A calculated expression cannot fix it. It reads the record's own fields, and the precision argument
of `round` is a number, while the digits a line needs live two hops away - on its document's
currency. Every module ends up writing `round(Quantity * Price, 2)`.

## The proposed shape

```yaml
entities:
  - name: Currency
    fields:
      - { name: code,     type: string, length: 3 }
      - { name: rounding, type: integer }            # the minor units: 0, 2, 3
  - name: SalesInvoice
    fields:
      - { name: total, type: decimal, scale: 4, aggregate: true, roundTo: Currency.rounding }
    relations:
      - { name: Currency, kind: manyToOne, to: Currency }
  - name: SalesInvoiceItem
    fields:
      - { name: net, type: decimal, scale: 4, roundTo: SalesInvoice.Currency.rounding,
          calculatedOnCreate: "Quantity * Price", calculatedOnUpdate: "Quantity * Price" }
    relations:
      - { name: SalesInvoice, kind: manyToOne, to: SalesInvoice, composition: true }
```

## Expected behaviour

- `roundTo:` on a `decimal` field names a path of to-one relations off the record, ending in an
  integer field: the number of decimal places the amount is stated to.
- The stored value is rounded half up to that many decimals: a value the caller sends, a calculated
  value and a document total resummed from its lines alike.
- A calculated value is evaluated directly to those decimals, never first to the column's scale and
  then again, so 0.495 in a currency of no minor units is 0, not 1.
- Checks, roll-ups and aggregates read the rounded value.
- The digits never exceed the column's `scale`: a column holds no more than its scale. A record whose
  path reaches nothing (no currency chosen, or a currency without a rounding) keeps the column's
  `scale`.

## Edge rules

- Valid only on a `decimal` field that is not the primary key.
- Every segment but the last is a to-one relation; a relation to another model may only be the last
  relation of the path. The last segment is an integer field; a terminal on another model's entity
  is trusted.
- A file that does not resolve, ends in a relation or in a non-integer field is rejected.
- A value with more decimals than the currency allows is rounded, not refused, as a value with more
  decimals than its column's scale is.

## Prior art / workarounds

A literal precision in every expression (`round(x, 2)`), wrong for every currency but the one it was
written for. Reference implementation: https://github.com/eclipse-dirigible/dirigible/pull/8837.

## Specification text

**Anchor:** Entities & fields > fields, a row in the field-key table after `precision` / `scale`, and
the paragraph below appended after that table.

| `roundTo` | a decimal amount's decimals read off the record: a to-one path ending in an integer field (`Currency.rounding`) |

A money amount is stated in a currency, and a currency has its own number of decimals. `roundTo:`
names where the record finds them.

> **Normative.**
> A conforming generator MUST accept `roundTo:` only on a `decimal` field that is not the primary
> key, and only as a path of to-one relations from the record ending in an integer field, a relation
> to another model being at most the last relation. It MUST store the field rounded half up to the
> number of decimals that path reads, but never to more than the field's `scale`, and to the `scale`
> when the path reaches no value. The rule applies to a value the caller writes, to a calculated value
> - which MUST be evaluated directly to those decimals - and to a document total. Checks, roll-ups and
> aggregates MUST read the rounded value. A value with more decimals is rounded, not refused.

## DSL index

| Construct | What it does |
| --- | --- |
| [`fields`](#fields) `roundTo` | a decimal rounded to its currency's minor units |
