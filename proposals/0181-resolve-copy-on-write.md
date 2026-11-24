# A register lookup's copy is made on the record's own write, and the form shows it first

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7911
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8890

## The problem

`resolves:` with `copy:` was meant to price a line from the price list in force on its document's
date. Written down as a price list over products, it never prices a line a person enters:

```yaml
entities:
  - name: SalesInvoiceItem
    fields:
      - { name: price, type: decimal, required: true,
          dependsOn: { relation: Product, valueFrom: price } }   # the form's default
resolves:
  - name: priceFromList
    event: { onCreate: SalesInvoiceItem }
    set: priceListItem
    from: PriceListItem
    match: { product: product, priceList: salesInvoice.customer.priceList }
    between: { start: validFrom, end: validTo, value: salesInvoice.issuedOn }
    copy: { price: price }
```

A copy is written after the record is stored, in reaction to its creation, and only into a field
that is still empty. Every path that matters defeats one of the two halves:

- The form fills the price first, from the product, so it is never empty when the lookup runs. Removing
  that default leaves the form with no price at all until after the save, so a person saves a line
  whose amount they have not seen.
- A `required: true` price left empty refuses the write before the lookup can run.
- Where the field is optional, the copy lands after the record's calculated fields were computed:
  the line's amount and the document's total stay computed from no price.

## The proposed shape

No new key. The meaning of `copy:` changes in when it is applied:

- on the record's own write - create for an `onCreate` lookup, update for an `onUpdate` one - before
  the record's calculated fields and checks read it;
- and, in a generated form, when the copied field's `dependsOn` fires, so the value shown before the
  save is the register's.

## Expected behaviour

- **A line created without a price is stored priced**, by whatever path creates it - a form, an
  interface, an import. Its calculated amount reads the copied price, a `required: true` check is
  satisfied by it, and a document total that sums the line includes it.
- **The form shows the register's value before the save.** When a copied field's `dependsOn` fires,
  the form asks the lookup for the record as it holds it and shows the covering row's value; the
  `dependsOn` value is the fallback when no row covers, or more than one does. The value shown is the
  value stored.
- **Nothing a record carries is replaced.** A price a person typed, a price a create-from copied from
  its source line, and a field's own `defaultValue:` all stand.
- **The relation and the outcome are unchanged**: the covering row is linked and the outcome stamped
  as before, after the write.

## Edge rules

- The copy follows the lookup's event and its `when` guard: an `onCreate` lookup does not copy on an
  update.
- The copy reads the record after its declared defaults, because a match key may be a defaulted
  relation. A copied field that declares its own `defaultValue:` therefore carries a value and is not
  copied into on a create that leaves it out.
- A copied field without a `dependsOn` is filled on the save; the form does not ask before it.

## Prior art / workarounds

Keeping the product's own columns as the line's default (a wholesale and a retail price per product,
picked by the customer's list) and not using a price list at all. Replacing a value equal to the
form's default on the server was the alternative considered: it needs the form's default recomputed on
the server, still shows a person a different price than the one stored, and replaces a typed value
that happens to equal the default.

## Specification text

**Anchor:** Declarative glue > resolves — fill a relation from a register valid on a date, replacing
the second sentence of the last normative block, and adding a paragraph and a normative block after
the normative block about `copy:`.

The last normative block's second sentence reads:

> The resolved relation and the outcome MUST be written as one targeted write of those columns only,
> leaving every other column of the record — and any concurrent write to it — untouched.

and the following is added after the normative block about `copy:`:

A copy is part of the record, not a correction made to it afterwards. It is made on the record's own
write, so a line's amount, a required-value check and a document total see it, whichever path wrote
the line; and a form asks for it before it shows the field, so the price a person sees is the price
the line is stored with.

> **Normative.**
> A copy MUST be made on the write the lookup's event names - the create for `onCreate`, the update
> for `onUpdate` - when the `when` guard holds, after the record's declared defaults and before its
> calculated fields and checks are evaluated, so that they and any total over the record read the
> copied value. It MUST NOT replace a value the record carries, including a field's declared
> default. A generated form that pre-fills a copied field from its `dependsOn` MUST instead show the
> value the lookup would copy for the record as the form holds it, falling back to the `dependsOn`
> value when the lookup reports `notFound` or `ambiguous`. A record MUST NOT be observable pointing at
> a covering row whose declared copies have not landed.

## DSL index

No new construct. The row of `resolves[].copy` reads:

| Construct | What it does |
| --- | --- |
| [`resolves[].copy`](#resolves--fill-a-relation-from-a-register-valid-on-a-date) | scalars of the covering row written onto empty record fields on the record's own write, and shown by the form before the save |
