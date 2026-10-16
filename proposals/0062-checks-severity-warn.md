# A soft tier of checks - `severity: warn`, `duplicate` and `itemsCompare`

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/98, https://github.com/eclipse-dirigible/dirigible/issues/7466, https://github.com/eclipse-dirigible/dirigible/issues/7524
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7513, https://github.com/eclipse-dirigible/dirigible/pull/7527

## The problem

Every check kind is a hard refusal. Most real rules about a name or a line sit in between: warn
before a second customer with the same name - a hard `unique:` is wrong, two companies may share a
registered name under different registration numbers - and warn before saving a document with
zero-value lines. The person who reviewed a billing application against the systems it replaces asked
for exactly this tier three times, and drew the line: a hard refusal for a second customer with the
same registration number and for forbidden characters in a name; everything else a confirmable
warning, one prompt per save.

Today the author chooses between refusing and saying nothing, and a model that says nothing lets the
duplicate in.

## The proposed shape

```yaml
- name: Customer
  checks:
    - { kind: duplicate, fields: [name],
        message: "A customer with this name already exists: {match}" }
    - { kind: compare, field: discount, op: le, value: 50, severity: warn,
        message: "A discount above 50%" }
- name: SalesInvoice
  checks:
    - { kind: itemsCompare, field: price, op: gt, value: 0,
        message: "{count} line(s) at price zero" }
```

## Expected behaviour

A warning does not refuse the write. A conforming implementation evaluates the write's warnings before
storing anything, and when any holds it answers the caller with the list - each a code and the
authored message - **without** storing. The same write, re-submitted with the caller's confirmation of
those warnings, is stored. A generated form shows the list and asks once; an API client is told the
same thing, so the tier is a rule of the record, not of one surface.

`severity: warn` is accepted on the ungated row-level kinds - `compare`, `requiredWhen`,
`forbidWhen`, `exactlyOne`. Two kinds exist only as warnings and take no `severity`:

- `duplicate` - another row of the entity carries the same values in `fields`. A `string` / `text`
  member compares ignoring case and surrounding spaces, so "ACME Ltd" and " acme ltd" collide.
  `{match}` in the message names the colliding record by its display label.
- `itemsCompare` - a line of the document's items child compares against a literal the way
  `compare` does; `{count}` in the message is the number of lines that match.

## Edge rules

- A gated check (`status:`) is never a warning: a transition either happens or does not. `severity`
  with `status` is refused.
- `duplicate` over a field that is not `string`, `text`, an integer or a to-one key is refused;
  normalisation applies to the text types only.
- A warning the caller confirmed is not re-asked within that write. Nothing is remembered across
  writes: the next edit that still trips it asks again.
- The hard kinds are unchanged: a refusal and a warning on one write answer with the refusal.

## Prior art / workarounds

The rendered site already documents this tier (IntentFile/intentfile.github.io#63) without a proposal
here; this proposal is what the site page should follow. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/7513 (the tier), https://github.com/eclipse-dirigible/dirigible/pull/7527 (normalised `duplicate`, `{match}`).

## Specification text

**Anchor:** Entities & fields > checks - declarative validations, as a new `####` subsection
`Warn and confirm - severity: warn, duplicate, itemsCompare`, after the last `kind:` subsection.

#### Warn and confirm - `severity: warn`, `duplicate`, `itemsCompare`

A check may warn instead of refuse. A warning is shown, and the write proceeds once the person
confirms it; it is enforced on every surface, not only on a form.

```yaml
- name: Customer
  checks:
    - { kind: duplicate, fields: [name], message: "A customer with this name already exists: {match}" }
    - { kind: compare, field: discount, op: le, value: 50, severity: warn, message: "A discount above 50%" }
- name: SalesInvoice
  checks:
    - { kind: itemsCompare, field: price, op: gt, value: 0, message: "{count} line(s) at price zero" }
```

`severity: warn` turns an ungated row-level check into a warning. `duplicate` warns when another
row carries the same `fields`, comparing `string` / `text` values ignoring case and surrounding
spaces, with `{match}` naming the colliding record; `itemsCompare` warns about the document's lines
that satisfy the comparison, with `{count}` as their number. Neither takes a `severity`.

> **Normative.**
> A conforming generator MUST evaluate a write's warnings before it stores anything, MUST answer a
> write that raises one or more unconfirmed warnings with the full list - a stable code and the
> authored message each - without storing, and MUST store the same write when the caller confirms
> those warnings. The tier MUST be enforced on every generated write surface. A warning MUST NOT be
> promoted to a refusal and a refusal MUST NOT be demoted; a write that raises both is answered with
> the refusal. `severity: warn` MUST be accepted on `compare`, `requiredWhen`, `forbidWhen` and
> `exactlyOne` without a `status:` gate, and MUST be rejected with one, on `duplicate`, on
> `itemsCompare`, and on every other kind. A `duplicate` MUST compare `string` / `text` members
> ignoring case and leading and trailing whitespace and every other member by value, MUST exclude
> the row being updated from its own search, and MUST resolve `{match}` to the colliding record's
> display label. An `itemsCompare` MUST resolve `{count}` to the number of matching lines.

## DSL index

| Construct | What it does |
| --- | --- |
| [`severity: warn`](#warn-and-confirm--severity-warn-duplicate-itemscompare) | an ungated row-level check that warns and asks for confirmation instead of refusing, enforced on every surface |
| [`checks: kind: duplicate`](#warn-and-confirm--severity-warn-duplicate-itemscompare) | warn before a second row with the same values; text compared ignoring case and surrounding spaces; `{match}` names the collision |
| [`checks: kind: itemsCompare`](#warn-and-confirm--severity-warn-duplicate-itemscompare) | warn about the document's lines that satisfy a comparison; `{count}` is their number |
