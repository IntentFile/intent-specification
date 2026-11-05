# A gated comparison reads another record at the move - `compare` `than:` over a to-one relation

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8494
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8528

## The problem

A VAT protocol is drafted from a purchase invoice and stores the invoice's net less what its credit
notes credited. A credit note approved after the draft changes that figure, and the Issue then
declares the stale base. A status-gated check can compare an own field with a literal or with
another own field, and a one-hop field only with a literal. Nothing can say "the figure this
document stored still equals what its source record says now". The workaround - a rule on the
record's own status change - is told only that the status changes, so it refuses the Cancel as well.

## The proposed shape

```yaml
entities:
  - name: VatProtocol
    checks:
      - { kind: compare, field: base, op: eq, than: "PurchaseInvoice.net - PurchaseInvoice.credited",
          status: ISSUED, message: "The purchase invoice changed since this protocol was drafted" }
    relations:
      - { name: PurchaseInvoice, kind: manyToOne, to: PurchaseInvoice, model: purchases }
      - { name: Status, kind: manyToOne, to: ProtocolStatus, function: EntityStatus, init: DRAFT }
```

## Expected behaviour

- With a `status:` gate, a `compare`'s `than:` may be a formula over the fields of ONE to-one
  relation of the entity, local or owned by another model.
- The related record is read when the record is written carrying the gate status: the move into it.
  A Cancel to another status is never asked.
- A refusal is the check's refusal, with the authored message.
- The formula's result is rounded to the left field's scale before the comparison.
- A record that points at no related record has no right-hand side, which is not a violation.

## Edge rules

- The formula takes numbers, `+ - * /`, parentheses, `round`, `abs`, `min`, `max`, `ceil`, `floor`
  and `<relation>.<field>` terms; every term reads the same relation.
- Refused without a `status:` gate, on a left field that is not a number, and on a term that is not a
  number field of the related entity (when that entity is known to the model).
- As for every gated check, a later write of a record still in the gate status is judged again.

## Prior art / workarounds

A rule on the record's own status change, which also refuses the moves it is not about. Reference
implementation: https://github.com/eclipse-dirigible/dirigible/pull/8528.

## Specification text

**Anchor:** Entities & fields > checks - declarative validations > kind: compare - two values of
one row. The `than` bullet becomes:

- `than` — the right operand as another field of the same record, or, with `status`, a formula over
  one to-one relation of the record (see below). Exactly one of `than` / `value`.

**Anchor:** the same section, a new subsection after its normative block:

##### Against another record - a gated `than:` formula

A gated `compare` MAY name, as `than:`, a formula over the fields of one to-one relation of the
entity, which MAY be a cross-model relation. It states "the figure this document stored is still
what its source says" at the move that relies on it:

```yaml
- { kind: compare, field: base, op: eq, than: "PurchaseInvoice.net - PurchaseInvoice.credited", status: ISSUED,
    message: "The purchase invoice changed since this protocol was drafted" }
```

The related record is read when the record is written carrying the gate status; a move into another
status - a Cancel - is never asked. The formula's result is rounded to the left field's scale, and a
record that references no related record has no right-hand side. As for every gated check, a later
write of a record still carrying the gate status is judged again, against the related record as it
then stands.

> **Normative.**
> A `than:` that reads a related record MUST be refused without a `status:` gate, on a left field
> that is not a number, when its terms read more than one relation, and when it contains anything but
> numbers, `+ - * /`, parentheses, the functions `round`, `abs`, `min`, `max`, `ceil`, `floor` and
> `<relation>.<field>` terms. A term that is not a number field of the related entity MUST be refused
> wherever the generator knows that entity. A conforming generator MUST read the related record when
> the record is written carrying the gate status, MUST NOT evaluate the check on a write into another
> status, and MUST round the formula's result to the left field's scale before it compares. A record
> that references no related record MUST NOT be refused by the check.

## DSL index

A new row, after `checks: kind: compare`:

| Construct | What it does |
| --- | --- |
| [`checks: compare` `than: "<relation>.<field> ..."`](#against-another-record---a-gated-than-formula) | a gated comparison against a formula over one related record, read at the move into the gate status |
