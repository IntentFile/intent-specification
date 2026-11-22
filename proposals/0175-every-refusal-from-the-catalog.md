# Every refusal a person reads is a translated catalog entry

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8775
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8850

## The problem

A generated application refuses a request in two ways. A `checks:` message and the refusal of one
field's value (required, too long, off its pattern, a repeated unique value) come from the
application's message catalog: translated, naming the field by its label, and carrying the key and
the field, so a form marks the property. Every other refusal a person reads was a sentence in the
default language, naming the entity, the action and the property by their identifiers and a status
by its id:

- `Approve is allowed only from status [3, 4] - current status is [6]`
- `Currency.code is required`
- `This Invoice cannot move from Draft to Paid`

A reader of another language read these in the default one, with technical names, and the form
could not mark the property.

## The proposed shape

No new key. Every refusal the generated application answers a person with is a catalog entry -
including the ones the file does not author: the status guards of `transitions:` and of a
create-from, a `lifecycle:` move, a record locked by `immutable:` / `immutableWhen` /
`immutableInPeriod`, the lock a master passes to its details, a status only the workflow moves, a
`fixedAfterCreate` / `fixedWhenReferenced` field, a roll-up capacity, a `leafOnly` reference, a
hierarchy cycle, a missing value of a required column, a share link that cannot be made, and a
create-from's "already generated" / "nothing to undo" refusals.

```yaml
rollups:
  - name: fundAllocated
    entity: PledgePayment
    via: Fund
    field: allocated
    op: sum
    of: amount
    capacity: budget
    message: "This fund has only {remaining} left of {capacity}; this allocation asks {requested}."
```

## Expected behaviour

- Each refusal is an entry of the model's catalog under `checks.<key>`. A sentence every model
  shares (a required value, a status move, a locked record) has one fixed key and a default text
  the generator writes into the default-language catalog for a model that declares the feature it
  refuses; an authored sentence (a roll-up `message:`, a `reversals: refuse:` message, an agree
  check seen from the record it links) has a key of its own.
- A refusal names an entity, an action and a property by its catalog label, and a status by the
  label of its row as the reader's language reads it - translated when the status entity is
  `multilingual`.
- An authored roll-up `message:` keeps `{capacity}`, `{sum}`, `{requested}` and `{remaining}` as
  ordinary placeholders, filled after translation, so a translation may move them.
- A refusal about one property carries that property as its field.
- The answer of a transition and of a create-from to a refused run carries the same message, key,
  parameters and field as every other refusal.

## Edge rules

- A shared sentence is resolved from the application's own catalog in the reader's language first,
  then from a translation the runtime ships for every application, then from the default text. An
  application without a catalog of the reader's language still refuses in that language when the
  runtime ships it.
- An entity, a status or an action another model owns is named by the label that model gives it
  where the runtime can reach it, and by its humanized name otherwise.
- Not covered: refusals a person does not read as a sentence (not found, forbidden, method not
  allowed), errors addressed to a developer (a malformed filter, a malformed upload) and answers of
  external systems.

## Prior art / workarounds

Before this, a translator could translate the field refusals and the authored `checks:` messages
only; every other refusal stayed in the default language. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/issues/8775.

## Specification text

**Anchor:** Constraints > checks, appended after the paragraph on translated check messages.

Every refusal the generated application answers a person with is an entry of the model's catalog,
under `checks.<key>`, and is resolved for the reader's language exactly as a check's message is:
the status guards of transitions and create-froms, lifecycle moves, locked records and the lock a
master passes to its details, workflow-owned statuses, fixed fields, roll-up capacities, hierarchy
references, required values and the refusals of a create-from's undo.

> **Normative.** A refusal names an entity, an action and a property by its catalog label and a
> status by the label of its row in the reader's language. A refusal about one property carries
> that property as its field. A transition's and a create-from's refusal carry the same message,
> key, parameters and field as every other refusal. An authored roll-up `message:` is a catalog
> entry whose `{capacity}`, `{sum}`, `{requested}` and `{remaining}` are placeholders filled after
> translation.

A sentence shared by every model is resolved from the application's catalog in the reader's
language, then from the translation the runtime ships, then from its default text.
