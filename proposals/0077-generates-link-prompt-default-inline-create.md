# A create-from records a link row to its target, a prompt input takes a default, and a to-one may opt out of inline creation

- **Status:** released in [1.12](../versions/1.12.md)
- **Issue:** https://github.com/IntentFile/intent-specification/issues/138, https://github.com/eclipse-dirigible/dirigible/issues/7748
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7756

## The problem

A clerk wants to record a payment **against this invoice**. 1.11 knows `generates.prompt` (a dialog
for what the source cannot derive) and, separately, `settlements` (automatic allocation of a payment
over open documents), but nothing that joins them. The only way to a payment from an invoice was the
allocation row's picker and its generated inline "New" dialog, which commits the payment on its own.
That payment's create event reached the automatic allocation first, which spread it over the
customer's oldest open documents; the allocation row being entered was then refused by its capacity
guard although the payment had been saved. The clerk was told the save failed while invoice #1 stood
PAID and invoice #2 PARTIAL.

Three gaps make up the one business act:

- a create-from cannot write a **link row** (the allocation) between its source and the created
  target inside the same unit of work;
- a prompt input has no **default** read from the source (the invoice's balance);
- a to-one relation cannot switch off the generated inline "New" that opens the back door.

1.11 also restricts the prompt: "the target MUST be a local entity (no `uses:`) declaring a
composition to-one relation to `forEntity`". The payment is neither local nor a child of the invoice.

## The proposed shape

```yaml
generates:
  - name: record-payment
    from: SalesInvoice
    to: CustomerPayment
    uses: customer-payments                    # the target may be another model's
    map: { Customer: Customer, Currency: Currency }
    defaults: { date: now }
    prompt:
      - { field: amount, required: true, default: balance }   # default: a property of the SOURCE
      - { field: Method, required: true }
    link:
      entity: SalesInvoiceCustomerPayment      # a composition child of the source
      map: { amount: amount }                  # link field <- created target's field
      # relation: CustomerPayment              # which to-one of the link points at the target, when ambiguous

entities:
  - name: SalesInvoiceCustomerPayment
    relations:
      - { name: CustomerPayment, kind: manyToOne, to: CustomerPayment, inlineCreate: false }
```

## Expected behaviour

**`link:`** - after the target and its lines are written, and inside the same unit of work, the
create-from writes one row of the named entity, a composition child of the **source**: its
composition relation is set to the source, its to-one to the target is set to the created target's
key, and `map` copies values from the created target onto it. The link row is written through the
link entity's own write path, so the guards that path enforces (a roll-up capacity, status-gated
checks, NOT NULL, the period lock) apply. A refused link row takes the target back with it, and the
caller gets a 400. Because the events of the unit dispatch only after it commits, an automatic
allocation that reacts to the target's create finds it already allocated and does nothing.

**`prompt[].default:`** - names a property of the **source** record. The dialog pre-fills the input
from the record, and the endpoint assigns the source's value to an input left empty **before** it
enforces `required`, so a client that sends no value still gets the default. An explicit value wins.

**`prompt:` on any target** - the restriction to a local composition child of `forEntity` is lifted:
a standalone target and a target owned by another model are prompted too, each control typed from
the target's own definitions (its widget and caption; for a to-one, the picker over its target).

**`inlineCreate: false`** on a to-one relation - the relation's picker offers no inline "New" on the
record form, the document header or the document item dialog.

## Edge rules

- `link.entity` MUST be a local entity declaring a composition to-one to `from`; it MUST declare
  exactly one to-one to `to`, or `relation:` names which one; `link.map` copies between fields of the
  same type; the source MUST be local, so a create-from whose source another model owns takes no
  `link`. Each is an authoring error.
- There is no `link.target:` key: the target is the create-from's `to`, so the target-side relation
  is derived and `relation:` is only the disambiguator.
- Only what the link entity's own write path enforces guards the link row. A row check decided at the
  endpoint only - `compare`, `agree`, `forbidWhen`, a `requiredWhen` without a status - does not run
  for it. This is stated as a limit.
- For a target owned by another model, the mapped and prompted properties are checked against the
  owning model at generation.
- `prompt[].default` names a property of the source whose type matches the prompted field's.
- `inlineCreate: false` on a collection or a subset relation is an authoring error.
- A create-from without `link`, a prompt without `default` and a relation without `inlineCreate`
  read and generate as in 1.11.

## Prior art / workarounds

The allocation row's picker and its inline "New", which raced the automatic allocation as described
above; or recording the payment first and allocating it by hand afterwards. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/7756.

## Specification text

**Anchor:** Declarative glue > generates - create-from > Prompted input - `prompt:`, amended; then a
new subsection after it; and Relations & multi-model > relations > Relation attributes, appended.

*Prompted input - `prompt:`* - the constraints bullet is **amended** to read:

- Constraints: the action's scope MUST be `entity`; a `timestamp` field MUST NOT be prompted. The
  target MAY be any entity - a composition child of `forEntity`, a standalone entity, or an entity
  another model owns (`uses:`); each control is typed from the target's own definitions, a to-one
  as a picker over its target, and for a target another model owns the prompted properties are
  checked against the owning model at generation.

and gains a bullet:

- **`default:`** names a property of the **source** record whose type matches the prompted field's.
  The dialog pre-fills the input from the clicked record, and an input left empty takes the source's
  value; an explicit value wins.

> **Normative.** A prompt entry's `default` MUST be applied by the generated endpoint as well as by
> the dialog: an input the request leaves empty MUST take the source record's value of the named
> property, and `required` MUST be enforced after that assignment, so a request that sends no value
> for a defaulted input is not refused for it. A value the request carries MUST win over the default.

#### A link row - `link:`

A create-from MAY declare `link:` - one row of a composition child of the **source**, written
between the source and the target the create-from has just created. The canonical case is recording
a payment from the invoice it pays: the payment is the target, and the allocation that ties it to the
invoice is the link row.

```yaml
generates:
  - name: record-payment
    from: SalesInvoice
    to: CustomerPayment
    uses: customer-payments
    map: { Customer: Customer, Currency: Currency }
    prompt:
      - { field: amount, required: true, default: balance }
    link:
      entity: SalesInvoiceCustomerPayment      # a composition child of the source
      map: { amount: amount }                  # link field <- created target's field
      # relation: CustomerPayment              # names the to-one to the target when there are several
```

The link row's composition relation is set to the source, its to-one to the target to the created
target's key, and `map` copies values from the created target onto it. The target is the
create-from's `to`, so the link declares no target of its own; `relation:` only says which to-one of
the link entity points at it when more than one does.

> **Normative.** A conforming generator MUST write the link row after the target and its lines,
> inside the same unit of work, through the link entity's own write path, so that the guards that
> path enforces apply to it. A refused link row MUST take the target back with it - nothing the
> create-from wrote remains - and the caller MUST receive a validation failure. Events of the unit
> MUST dispatch only after the unit commits, so that a reaction to the target's create observes the
> link row. A row check decided only at an entity's own endpoint is not part of its write path and
> does not guard the link row.
>
> Each of the following MUST be reported as an authoring error: a `link.entity` that is not a local
> entity declaring a composition to-one to `from`; a link entity declaring no to-one to `to`, or more
> than one without `relation:` naming which; a `link.map` between fields of different types; and
> `link` on a create-from whose source another model owns.

*Relation attributes* - appended:

- **`inlineCreate: false`** on a to-one relation removes the inline "New" from every picker generated
  for it - the record form, the document header and the document item dialog - so a target row is
  created only through its own surfaces or a declared create-from. The default is `true`.

> **Normative.** A conforming generator MUST NOT offer inline creation of the target in any picker
> it generates for a to-one relation declaring `inlineCreate: false`. `inlineCreate` on a relation
> that is not a to-one, and on a subset relation, MUST be reported as an authoring error.

## DSL index

| Construct | What it does |
| --- | --- |
| [`generates.link`](#a-link-row--link) | write one composition child of the source between it and the created target, in the same unit of work and through its own write path - the allocation that ties a recorded payment to its invoice |
| [`generates.prompt[].default`](#prompted-input--prompt) | pre-fill a prompted input from a property of the source; an input left empty takes it at the endpoint too |
| [`relations` / `inlineCreate`](#relation-attributes) | `false` removes the inline "New" from a to-one's pickers |

The `generates.prompt` row gains "on any target, local, standalone or another model's".
