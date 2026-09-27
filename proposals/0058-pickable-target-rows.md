# Which target rows a picker offers

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7496
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7514

## The problem

A to-one relation is filled from a picker listing the target's rows, and every row is offered the
same way. Most business rules about WHICH row may be picked are not a filter on one value; they are
a completeness rule over the target row.

```yaml
- name: SalesInvoice
  relations:
    - { name: Customer, kind: manyToOne, to: Customer, model: customers, required: true }
```

An invoice can only go out to a customer who carries a registration number and an address. A
customer created in a hurry with only a name is a legitimate row - it may already have quotations,
and its data will be completed later - so it cannot be deleted or refused at creation. Nothing in
the model says it cannot be invoiced yet. A clerk picks it onto an invoice, types twenty lines, and
learns only at the transition that issues the invoice that it cannot go out: the write-side check
refuses correctly, and far too late.

`where:` does not say it: it narrows the options by one static value, and a missing value is not a
value. `dependsOn.filterBy` does not say it either: it narrows by the value another control holds.
Neither can say "offer this row only while it is complete", and neither can say why a row is not
offered.

The same shape recurs across a suite: a supplier without bank details on a payment order, a product
without a unit of measure on an order line, an employee without a cost centre on a timesheet, an
account without a currency on a journal line.

## The proposed shape

A `pickable:` attribute on a to-one relation, naming the condition a target row must meet to be
picked and what the picker does with a row that does not:

```yaml
- name: SalesInvoice
  relations:
    - { name: Customer, kind: manyToOne, to: Customer, model: customers, required: true,
        pickable: { when: [registrationNumber != null, address != null],
                    else: mark, message: "Registration data incomplete" } }
    - { name: Product, kind: manyToOne, to: Product,
        pickable: { when: "active == true", else: hide } }
```

- `when:` is the condition, over the TARGET row: one term, or a list of terms, which is their AND.
  A term is `<target property> ==|!= <literal>` or `<target property> ==|!= null`.
- `else:` is what the picker does with a row the condition does not hold for: `mark` (the default)
  or `hide`.
- `message:` is what a marked row says about itself.

## Expected behaviour

- Under **`mark`** a failing row is listed, cannot be chosen, and shows the message next to its
  name, so the person picking learns why before choosing, not at the transition. Without an
  authored message the condition text itself is shown: a marked row always says why.
- Under **`hide`** a failing row is not listed.
- A value the record already holds is always shown with its label, and stays listed even when it
  now fails the rule - disabled, with the reason. A rule change, or a customer whose data was
  removed after the invoice was drafted, never turns a stored reference into an unlabelled key or
  silently empties the control.
- A picker whose options are narrowed by `dependsOn` or `where` applies the rule to what remains.
  Where the picker would select the only remaining option by itself, it does so only when exactly
  one remaining row is pickable.
- `== null` tests that the property is absent; `!= null` that it is present, meaning neither absent
  nor blank. A presence test works on a property of any type.
- The rule governs the PICKER only. A write that does not come through a picker - an API call, an
  import, an automated create - is not gated by it. The write-side rule is declared separately, as
  a `checks:` entry (typically a `requiredWhen` gated on the status that needs the value), and the
  two are meant to be declared together.

## Edge rules

- Only a `manyToOne` or `oneToOne` relation has a picker. `pickable:` is refused on any other kind,
  on a `subset` relation, on a composition parent (preset by the context that opens the record, and
  never picked), and on a `function: EntityStatus` relation (a read-only badge).
- `pickable:` together with `leafOnly` is refused: a hierarchy picker offers its own rows, and a
  rule it does not apply would read as enforced while it is not.
- A term reads the target row itself. Every property it names MUST be a field or a to-one relation
  of the target; a path (`country.code`) is refused.
- A value comparison is on a string, an integer or a boolean field, or a to-one's key - the types
  an equality is exact on. A decimal, a double or a date is refused for a value comparison; it can
  still be tested for presence. A literal that is not a value of the compared property's type is
  refused.
- Several terms are a list, never a conjunction written inside one string (`"a != null && b != null"`
  is refused, and the refusal points at the list form). A term without the comparison shape is
  refused rather than read as always true or always false.
- `when:` is required. `else:` accepts only `mark` and `hide`.
- A target owned by another model is held to the grammar when the model is read; its properties are
  checked against the owning model at generation, and a property that model does not declare is an
  error there.

## Prior art / workarounds

- A **filter on one value** (`where:`) with a hand-maintained flag on the target ("invoiceable")
  that someone has to keep in step with the fields it summarises - and the picker still cannot say
  why a row is missing, so the clerk looks for a customer who is plainly there and does not appear.
- A **write-side check alone**, which is correct and arrives at the wrong moment: after the record
  has been fully typed, at the transition that needs the value.
- A **hand-written picker** per document type, replacing the generated one for one rule.

## Specification text

**Anchor:** Relations & multi-model > relations > Relation attributes, as a new bullet and example
after the bullet "**`where`** filters the dropdown to options matching a static condition." and
before the normative paragraph on `whenMasterDeleted`.

- **`pickable`** decides which of the TARGET's rows the relation's picker offers, and what it does
  with a row that does not qualify:

```yaml
- { name: Customer, kind: manyToOne, to: Customer, model: customers,
    pickable: { when: [registrationNumber != null, address != null],
                else: mark, message: "Registration data incomplete" } }
- { name: Product, kind: manyToOne, to: Product, pickable: { when: "active == true", else: hide } }
```

  `when` is one term or a list of terms, which is their AND; a term is
  `<target property> ==|!= <literal>` or `<target property> ==|!= null`, over a field or to-one
  relation of the target row itself. `== null` tests that the property is absent, `!= null` that it
  is present - neither absent nor blank. `else: mark` (the default) lists a failing row, not
  choosable, with `message` - or, absent a message, the condition text - shown next to its name;
  `else: hide` leaves it out. A value the record already holds is always shown with its label and
  stays listed, not choosable, when it fails the rule.

  `pickable` governs the picker, not the write: declare the write-side rule as a
  [`checks`](#checks--declarative-validations) entry, typically a
  [`requiredWhen`](#kind-requiredwhen--a-value-required-only-under-a-condition) gated on the
  status that needs the value.

> **Normative.** A conforming generator MUST apply a `pickable` rule to every picker it generates
> for the relation, including a picker narrowed by `dependsOn` or `where`, and MUST NOT offer a row
> that fails the rule as a choice. Under `else: mark` (the default) a failing row MUST be listed as
> not choosable, together with the authored `message`, or the authored condition when no message is
> given. Under `else: hide` a failing row MUST NOT be listed. A value the record already holds MUST
> be displayed with its label even when it fails the rule, and MUST remain listed, not choosable.
> Where a picker would select the only remaining option automatically, it MUST do so only when
> exactly one remaining row is pickable. `== null` MUST hold when the property is absent, and
> `!= null` when it is neither absent nor blank; a presence test MAY name a property of any type. A
> value comparison MUST be on a string, integer or boolean field or a to-one relation's key, and a
> literal that is not a value of the compared property's type MUST be reported as an authoring
> error. `pickable` MUST NOT be read as a constraint on writes: a write that does not come through a
> picker is not governed by it.
>
> Each of the following MUST be reported as an authoring error: `pickable` on a relation that is not
> a `manyToOne` or `oneToOne`, on a `subset`, on a composition parent or on a `function: EntityStatus`
> relation; `pickable` together with `leafOnly`; a missing `when`; an `else` other than `mark` or
> `hide`; a term that names a path, or a property that is not a field or to-one relation of the
> target; and a term without the comparison shape, which MUST NOT be read as always true nor as
> always false. For a target owned by another model the properties MUST be checked against the
> owning model at generation.

## DSL index

| Construct | What it gives you |
| --- | --- |
| [`relations` / `pickable`](#relation-attributes) | which target rows a to-one's picker offers: a failing row is listed disabled with a message (`mark`) or left out (`hide`); the picker only, the write-side rule is a `checks` entry |
