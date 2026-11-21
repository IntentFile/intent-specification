# Checks judge what a calculated action derives; an action is a pure computation

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8000
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8833

## The problem

The create-time and update-time checks (`checks:`, `requiredWhen`, `duplicate`, the reference and
picker rules) judge "the record this write would store". A `calculatedOnCreate` /
`calculatedOnUpdate` expression is part of that record, but the text does not say whether a
`calculatedActionOnCreate` / `calculatedActionOnUpdate` call-out is. An implementation that derives
the actions only at the moment of storing, after the checks ran, judges the caller's value instead -
usually nothing. A supplier note whose supplier code an action reads off its bill is never asked the
`duplicate: [supplierCode, supplierNumber]` question on a create that left the code out, although
the stored row has it.

Running the action for the checks as well means it can run more than once per write, and for a write
that is then refused. The text does not say what an action may do.

## The proposed shape

No new key.

```yaml
- name: SupplierNote
  imports: |
    import example.notes.NoteSupplierAction;
  fields:
    - { name: supplierNumber, type: string, length: 30 }
    - { name: supplierCode,   type: string, length: 20, calculatedActionOnCreate: NoteSupplierAction }
  checks:
    - { kind: duplicate, fields: [supplierCode, supplierNumber], message: "Already recorded: {match}" }
```

## Expected behaviour

- The checks of a create or update judge the record with its calculated actions applied, as they
  judge it with its calculated expressions applied, in declaration order, an action still winning
  over an expression on the same slot.
- An action may be invoked more than once for one write, and for a write that is then refused. It
  computes a value from the record and changes nothing else: no write, no message, no number drawn
  from a series - a document number is the first-class `number:` attribute.

## Edge rules

- The checks judge a copy; the submitted record is not changed by the evaluation.
- A model declaring no calculated action generates as before.

## Prior art / workarounds

Repeating the action's logic as an expression, where one can say it; otherwise none.

## Specification text

**Anchor:** Entities & fields > `Calculated fields`, after the paragraph beginning "The
implementation lives in the project's custom (escape-hatch) folder".

A calculated action is part of the record a write would store, so the write's checks see what it
derives. It is therefore a computation, not a side effect.

> **Normative.**
> A conforming generator MUST evaluate the create-time and update-time checks of an entity against
> the record with its calculated actions applied, as it does with its calculated expressions, in
> declaration order, an action taking precedence over an expression on the same slot. The submitted
> record MUST NOT be changed by that evaluation. A calculated action MAY be invoked more than once
> for one write, and for a write that is then refused; an action MUST therefore compute its value
> from the record and change nothing else.

## DSL index

No new row.
