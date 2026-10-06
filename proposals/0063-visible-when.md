# Status-gated visibility of a field or a composition panel - `visibleWhen`

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/99, https://github.com/eclipse-dirigible/dirigible/issues/7502
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7566

## The problem

A composition child's panel renders on the document page unconditionally. A DRAFT sales invoice
therefore shows an empty payment-allocation panel, an empty copy panel and an empty reminder panel -
three panels about things that cannot exist before the invoice is issued. The same holds for a field
that has no meaning until a status: a `paidOn` date on a draft.

What exists gates something else. `forbidWhen` and `locksWithMaster` hide the add / edit / delete
affordances but leave the panel; `visibleTo` gates a field by **role**, not by state. There is no
way to say "this is not part of the record until it reaches this status".

## The proposed shape

```yaml
- name: SalesInvoice
  fields:
    - { name: paidOn, type: date, visibleWhen: "Status == PAID" }
  relations:
    - { name: allocations, kind: oneToMany, to: SalesInvoiceCustomerPayment, composition: true,
        visibleWhen: "Status != DRAFT" }
```

## Expected behaviour

`visibleWhen: "<condition>"` on a field, or on a composition child relation (written in the master's
terms), leaves the input, the read-only row or the whole panel out of the generated views until the
condition holds against the record the page shows. The condition is `requiredWhen`'s:
`<Property> ==|!= <literal>` over the record's own properties, a list meaning their AND, a status
named by its seeded name.

It is a presentation rule only. The API still reads and writes the property; nothing is deleted or
cleared when a record regresses to a status under which the panel is hidden - the rows are simply out
of view until the condition holds again.

## Edge rules

- On a composition child the condition names the **master's** properties, since it is the master's
  page that decides whether to render the panel.
- A property the record does not carry, a literal of the wrong type, or a condition that does not
  parse is an authoring error, as for `requiredWhen`.
- `visibleWhen` and `visibleTo` compose: both must allow the field for it to render.
- A hidden field that is `required` and carries no default is an authoring error: the form could
  never be completed from the status that hides it.

## Prior art / workarounds

A hand-edited generated page, lost on the next generation. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/7566.

## Specification text

**Anchor:** Entities & fields, as a new `###` section `Status-gated visibility - visibleWhen`,
after `### Role-scoped field visibility - visibleTo`.

### Status-gated visibility - `visibleWhen`

A field, or a composition child, may be declared part of the record only from a state on:

```yaml
- name: SalesInvoice
  fields:
    - { name: paidOn, type: date, visibleWhen: "Status == PAID" }
  relations:
    - { name: allocations, kind: oneToMany, to: SalesInvoiceCustomerPayment, composition: true,
        visibleWhen: "Status != DRAFT" }
```

The condition has the grammar of `requiredWhen`'s `when`. A field's input and read-only row, or a
child's whole panel, are left out of the generated views while it does not hold against the record the
page shows.

> **Normative.**
> A conforming generator MUST omit the field or the child panel from every generated view of the
> record while the condition does not hold, and MUST render it as soon as the condition holds. The
> rule is presentation only: the API MUST continue to read and write the property, and a record that
> moves to a state under which a panel is hidden MUST keep that panel's rows unchanged. On a
> composition child the condition MUST be read against the master. `visibleWhen` and `visibleTo`
> compose. A generator MUST reject as an authoring error a condition that does not parse, a property
> the record does not carry, a literal that is not a value of the property's type, and a `required`
> field without a default that is hidden under any state.

## DSL index

| Construct | What it does |
| --- | --- |
| [`visibleWhen`](#status-gated-visibility--visiblewhen) | a field or a composition panel is rendered only while a condition over the record's status or properties holds; presentation only |
