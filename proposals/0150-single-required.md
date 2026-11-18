# A default that always exists - `required: true` on a `single` flag

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8700
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8708

## The problem

`single` keeps a flag on **at most** one row of its scope: when another row takes the flag, the
previous holder gives it up. Nothing keeps it on **at least** one row. A write that sets the flag
`false` on its only holder is accepted, and so is the delete of the holder; afterwards no row holds
it.

These flags exist to provide a default - the company of a new document, the base currency that
conversions are stated in, the payment method a new document starts with. Once no row holds the
flag, every reader of the default has nothing to read.

A model cannot close the gap itself. A `checks:` rule sees only the value being written
(`base == false`), which is legitimate on every row except the only holder, and no check kind counts
sibling rows. A `forbidWhen` on delete closes the delete of the holder but also refuses the delete of
the last row, which must stay possible.

## The proposed shape

```yaml
- name: Currency
  fields:
    - { name: id,   type: integer, primaryKey: true, generated: true }
    - { name: code, type: string, length: 3 }
    - { name: base, type: boolean, single: { required: true } }

- name: BankAccount
  fields:
    - { name: id,        type: integer, primaryKey: true, generated: true }
    - { name: iban,      type: string, length: 34 }
    - { name: isDefault, type: boolean, single: { per: company, required: true } }
  relations:
    - { name: company, kind: manyToOne, to: Company }
```

## Expected behaviour

- **Update.** A write that sets the flag `false` on its current holder is refused while no other row
  of the holder's scope holds it. The refusal is an invalid-input refusal naming the flag field, with
  one message shared by every such flag, seeded in the translation catalog and naming the entity and
  the flag by their translated labels - so one translated entry answers it in the reader's language
  for every entity, as the other field refusals are. The flag still moves the ordinary way: setting it on
  another row demotes the holder.
- **Scope change.** A holder whose write moves it to another `per` scope gives the flag up in the scope
  it leaves; that write is refused the same way, unless the holder is the last row of that scope.
- **Delete.** A delete of the holder is refused the same way, unless it is the last row of its scope.
- **Create.** A row may be created without the flag, so an empty scope can get its first row. The
  flag is not set on it automatically.
- `required` combines with `per` and with either `onConflict`. A model declaring no `required` on a
  `single` flag generates exactly as before.
- **The `onConflict: refuse` conflict is translated the same way.** The refusal of a write that takes
  a flag another row of its scope holds stays a conflict (409), now naming the flag field, with one
  message shared by every such flag (and one for a `per:` flag, naming the scope relation) seeded in
  the translation catalog and naming the entity, the flag and the scope by their translated labels.
  It used to be an English sentence with those names written into it.

## Edge rules

- A row is the holder only when its stored flag is `true`; a write to any other row is not concerned.
- A delete that removes a whole scope together with its owner (a composition cascade) is not a
  delete of the holder by itself, and an implementation is not required to refuse it.
- As for `single` itself, an implementation is not required to apply the rule to a targeted write of
  the flag alone, nor to serialise two concurrent writes that each see another holder.

## Prior art / workarounds

`forbidWhen: ["base == true"], verbs: [delete]` closes the delete of the holder, at the cost of the
last row; nothing closes the update. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8708.

## Specification text

**Anchor:** Data model > fields > `single — at most one row holds the flag`: a paragraph and a
normative block after the existing normative block.

Paragraph:

`required: true` turns the default the flag marks into one that always exists: the holder cannot give
the flag up - by being written `false`, moved to another scope, or deleted - while no other row of its
scope holds it. The flag moves by being set on another row, as before.

```yaml
fields:
  - { name: base, type: boolean, single: { required: true } }
```

> **Normative.**
> For a `single` flag declaring `required: true`, a conforming generator MUST refuse, as invalid input
> naming the flag field and with a message seeded in the translation catalog that names the entity and
> the flag by their translated labels, a write that sets the flag `false` on the row that stores it
> `true`, and a write that moves that row to another `per` scope, unless another row of the scope it gives the flag up in holds the flag, or - for a move - no
> other row remains in that scope. It MUST refuse the delete of that row the same way unless no other
> row remains in its scope. It MUST NOT refuse a write that stores a row without the flag when the row
> is not the holder, including the creation of a scope's first row, and MUST NOT set the flag on it.
> For a `single` flag declaring `onConflict: refuse`, the refusal of a write that sets the flag while
> another row of its scope holds it MUST name the flag field and carry a message seeded in the
> translation catalog that names the entity, the flag and - for a `per` flag - the scope relation by
> their translated labels.
> A model declaring no `required` on a `single` flag MUST generate byte-identical output.

## DSL index

| Construct | What it does |
| --- | --- |
| [`single`](#single--at-most-one-row-holds-the-flag) `required` | the flag never leaves its scope without a holder: the holder can be neither un-flagged nor deleted until another row takes it |
