# Two relations of one row that must agree on a shared target - `checks: kind: agree`

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/97, https://github.com/eclipse-dirigible/dirigible/issues/7409, https://github.com/eclipse-dirigible/dirigible/issues/7589, https://github.com/eclipse-dirigible/dirigible/issues/7631
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7418, https://github.com/eclipse-dirigible/dirigible/pull/7626, https://github.com/eclipse-dirigible/dirigible/pull/7677

## The problem

A junction row links two documents, and the link is only meaningful when both documents agree on a
third thing. A payment allocation joins a sales invoice and a customer payment; an allocation whose
invoice belongs to one customer and whose payment belongs to another is not a disputed case, it is a
mistake, and so is one whose currencies differ. The rule is "the invoice's customer and the payment's
customer are the same record".

`compare` ([0035](0035-two-values-of-one-row-compared.md)) compares two values of one row, and reads
at most one hop on one side. The rule above reads one hop on **both** sides, so it cannot be written.
A second shape of the same rule has one hop on one side and none on the other: an opening balance
carries a fiscal `year`, which belongs to a company, and a `company` of its own; what must hold is
`Year.Company == Company`.

Without the check the allocation is saved, the invoice's `paid` roll-up moves, and the wrong
customer's balance is settled. The mistake surfaces at the next statement, far from where it was made.

## The proposed shape

```yaml
- name: SalesInvoiceCustomerPayment
  checks:
    - { kind: agree, relations: [SalesInvoice, CustomerPayment], onProperty: Customer,
        message: "This payment belongs to a different customer than the invoice" }
    - { kind: agree, relations: [SalesInvoice, CustomerPayment], onProperty: Currency,
        message: "The payment's currency does not match the invoice's" }

- name: OpeningBalance
  checks:
    # one hop on the left, the record's own to-one on the right
    - { kind: agree, relations: [year, company], onProperty: company,
        message: "The fiscal year belongs to another company" }
```

## Expected behaviour

`relations` names exactly two to-one relations of the record; `onProperty` names the property both
sides are compared on. Each side resolves as `<relation>.<onProperty>`, a one-hop read of the related
row. When one side's target carries no such property while the other side's does, that side falls back
to the record's **own** to-one named by the bare relation name, so `[year, company]` with
`onProperty: company` reads `year.company` against `company`.

The write of the junction row is refused with `message` when the two resolved values differ. A side
that resolves to no value - an unset relation, or a related row whose property is empty - is not a
disagreement: the check says nothing, and a `required` on the relation says the rest.

The check also holds from the other direction. Re-pointing the agreed property on either related
record - moving the payment to another customer - while a junction row still references it would
leave that row in disagreement, so the related record's write is refused with the same message.

## Edge rules

- Both sides MUST end on the same entity. Two keys of different nomenclatures - a company id against
  a customer id - are refused at generation: the comparison can only ever be false.
- `onProperty` that resolves on neither side is reported as such; it is never read as a comparison
  of the two bare foreign keys.
- The fallback to the record's own to-one is taken only when the other side's hop resolved.
- The check is ungated: it holds on every user write of the junction row. A `status:` key is refused.
- The related records' guard reads the junction rows that reference them at the time of their write;
  it refuses the write, it never cascades a correction.

## Prior art / workarounds

A hand-written validation in the project's custom code, one per junction, each re-implementing the
two reads and the message. Reference implementation: https://github.com/eclipse-dirigible/dirigible/pull/7418 (the kind),
https://github.com/eclipse-dirigible/dirigible/pull/7626 (the parents' guard), https://github.com/eclipse-dirigible/dirigible/pull/7677 (the own-to-one side).

## Specification text

**Anchor:** Entities & fields > checks - declarative validations, as a new `####` subsection
`kind: agree - two relations that must agree on a shared target`, after
`#### kind: compare - two values of one row`.

#### kind: agree - two relations that must agree on a shared target

An `agree` check names two to-one relations of the record and a property both of their targets
carry. It refuses the write while the two related records disagree on that property:

```yaml
- name: SalesInvoiceCustomerPayment
  checks:
    - { kind: agree, relations: [SalesInvoice, CustomerPayment], onProperty: Customer,
        message: "This payment belongs to a different customer than the invoice" }
- name: OpeningBalance
  checks:
    - { kind: agree, relations: [year, company], onProperty: company,
        message: "The fiscal year belongs to another company" }
```

Each side is read as `<relation>.<onProperty>`. Where one side's target has no such property and the
other side's has, that side is the record's own to-one named by the bare relation name - the shape of
a record that carries both a thing and the thing's owner.

> **Normative.**
> A conforming generator MUST refuse a create or update of the row, with `message`, when both sides
> resolve to a value and the values differ; a side with no value MUST NOT be read as a disagreement.
> It MUST also refuse a write of either related record that changes the agreed property while a row
> of the declaring entity still references it, with the same message. Both sides MUST resolve to the
> same target entity; two different nomenclatures MUST be reported as an authoring error at
> generation. An `onProperty` that resolves on neither side MUST be reported as such and MUST NOT be
> read as a comparison of the two foreign keys. `relations` MUST name exactly two to-one relations of
> the record; `agree` takes no `status:` gate.

## DSL index

| Construct | What it does |
| --- | --- |
| [`checks: kind: agree`](#kind-agree--two-relations-that-must-agree-on-a-shared-target) | two to-one relations of one row whose targets must agree on a property - a junction's invoice and payment on the customer; a side may be the record's own to-one |
