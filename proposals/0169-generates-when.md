# A create-from guarded by the source's own values

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7992
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8842

## The problem

A create-from can be guarded only by the source's status: `fromStatus:` lists the statuses the
source may stand in, and nothing else refuses the invocation or withholds its button. A source whose
eligibility is a plain field cannot be expressed. An invoice template whose lifecycle is a boolean
`active` keeps offering "Create invoice now" while it is paused, and the invocation creates an
invoice. The same holds for any field-shaped eligibility: a flag, a reference that must be set, a
category that must match.

## The proposed shape

Two optional keys on a `generates:` entry:

```yaml
generates:
  - name: invoice-from-template
    from: InvoiceTemplate
    to: SalesInvoice
    forEntity: InvoiceTemplate
    when: ["active == true", "Customer != null"]
    whenMessage: "Activate the template first"
    map: { Customer: Customer, Template: id }
```

`when` is a condition over the source record in the vocabulary a `forbidWhen` check uses for the
record's own values: one comparison, or a list of comparisons that must all hold; each compares a
field or to-one relation of the source with `==` or `!=` against a literal, a seeded status name, or
`null`. `whenMessage` is the text a refused invocation receives.

## Expected behaviour

- An invocation on a source for which `when` does not hold is refused with `whenMessage`, before
  anything is created and before a document number is drawn; it is the same kind of refusal as a
  `fromStatus` miss.
- The button is not offered on a record for which `when` does not hold.
- `when` and `fromStatus` combine: both must admit the invocation.

## Edge rules

- `when` without `whenMessage`, and `whenMessage` without `when`, are authoring errors.
- A comparison that names no field or to-one relation of the source, or whose literal is not a value
  of that property's type, is an authoring error.
- Like `fromStatus`, `when` guards an invocation by a user: it is an authoring error on a create-from
  that contributes no button (an event-driven one qualifies its moment with the event's own `when`),
  on an action scoped to the whole view, and on a scheduled generation (which selects its source rows
  with `where:`).
- An implementation MAY refuse `when` on a source owned by another model, whose fields it cannot type.
- Ordering comparisons (`>`, `<`) are not part of this vocabulary.

## Prior art / workarounds

A hand-written refusal in an extension hook, which leaves the button offered; or modelling the
eligibility as a status relation only so that `fromStatus` can read it.

## Specification text

**Anchor:** Declarative glue > `generates — create-from`, a new paragraph and normative block
directly after the normative block on `fromStatus`.

A create-from may also be guarded by the source's own values:

```yaml
generates:
  - name: invoice-from-template
    from: InvoiceTemplate
    to: SalesInvoice
    forEntity: InvoiceTemplate
    when: "active == true"            # a condition over the source record
    whenMessage: "Activate the template first"
```

> **Normative.**
> A create-from declaring `when` is refused, with its `whenMessage`, when the condition does not hold
> on the source record; the condition is read in the vocabulary of a `forbidWhen` check over the
> record's own fields and to-one relations (a comparison, or a list of comparisons that MUST all
> hold), and the refusal is decided before anything is created, against the source as it stands when
> the target is created. A per-record action is not offered on a record for which the condition does
> not hold, and the refusal on the action's own endpoint remains the enforcement. `when` and
> `fromStatus` both apply when both are declared. `when` without `whenMessage`, `whenMessage` without
> `when`, a comparison that does not read against the source, and `when` on a create-from that
> contributes no button, on an action scoped to the whole view or on a scheduled generation are
> authoring errors.

## DSL index

| Key | Where | Meaning |
| --- | --- | --- |
| `when` | `generates[]` | condition over the source record that must hold for the create-from to run |
| `whenMessage` | `generates[]` | the refusal an invocation receives when `when` does not hold |
