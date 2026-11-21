# A relation defaults to the row holding a flag - `init: { flag: }` on a to-one relation

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8683
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8707

## The problem

A to-one relation's `init:` names a fixed row: a seeded id, or the seeded name of a status. A
document's default company, base currency or default payment method is not a fixed row, though. It
is the row the user has flagged as the base one, through a [`single`](0091-single-flag.md) boolean.
A model has no way to say this, so it writes the id of whichever row happened to be first:

```yaml
- name: Company
  type: relation
  to: Company
  init: 1        # "the base company" - in fact whatever row has Id 1
```

Two things follow. Moving the flag to another company changes nothing: every new document still
defaults to row 1. And once row 1 is deleted, every new document starts with a reference to a row
that does not exist, and a generated key cannot bring it back. The same holds for every "base" or
"default" nomenclature a document defaults from.

## The proposed shape

```yaml
entities:
  - name: Company
    fields:
      - { name: id,   type: integer, primaryKey: true, generated: true }
      - { name: name, type: string, length: 100 }
      - { name: base, type: boolean, single: true }
  - name: GoodsReceipt
    fields:
      - { name: id, type: integer, primaryKey: true, generated: true }
    relations:
      - { name: Company, kind: manyToOne, to: Company, required: true, init: { flag: base } }
```

## Expected behaviour

- A create that leaves the relation unset takes the target row holding the flag at the moment of the
  create, in the same write. A create that sets the relation keeps its value.
- The create form preselects that row when it opens, so the user sees the default before saving and
  can change it.
- Moving the flag to another row moves the default for every record created afterwards. Records
  already created keep their reference.
- With no row holding the flag, the relation is left empty - never a fixed id. A `required:` relation
  is then refused as a missing value, naming the relation.
- An update that leaves the relation out keeps the stored value, as for any defaulted member.
- The relation's storage carries no fixed default value.
- A model declaring no `init: { flag: }` generates exactly as before.

## Edge rules

- `flag` names a field of the relation's target that is a `boolean` with a fleet-wide `single`. Any
  other field, an unknown field, or a mapping without `flag` is an authoring error. For a target in
  another model, the check is made against that model.
- A `single: { per: <relation> }` flag has one holder per scope, and resolving the scope of the record
  being created is not part of this proposal: naming one is an authoring error, never read as
  fleet-wide.
- `init: { flag: }` on a `function: EntityStatus` relation is an authoring error: a status starts at a
  seeded status.
- `init: { flag: }` is a key of `manyToOne` / `oneToOne` relations only, like the scalar `init`.

## Prior art / workarounds

`init: <id>` naming the first row, with the defects above, or a hand-written create-time action that
reads the flag. Reference implementation: https://github.com/eclipse-dirigible/dirigible/pull/8707.

## Specification text

**Anchor:** Relations & multi-model > relations > Relation attributes: a bullet after
`function: EntityStatus`.

Bullet:

- **`init: { flag: <field> }`** defaults a to-one relation to the target row holding the named
  [`single`](#single--at-most-one-row-holds-the-flag) boolean, instead of a fixed record: `- { name: Company, kind: manyToOne, to: Company, init: { flag: base } }`.

> **Normative.**
> For a to-one relation declaring `init: { flag: <field> }`, a conforming generator MUST, on a create
> that leaves the relation unset, set it to the target record whose `<field>` is `true` at the time of
> the create, within the same write, and MUST leave it unset when no target record holds the flag. It
> MUST NOT emit a fixed default for the relation's storage. A generated create form MUST preselect the
> holder when it opens. `<field>` MUST be a `boolean` of the target with a `single` that is not scoped
> by `per`; any other field, a scoped flag, a mapping without `flag`, and the form on a
> `function: EntityStatus` relation are authoring errors. A model declaring no `init: { flag: }` MUST
> generate as before.

## DSL index

| Construct | What it does |
| --- | --- |
| [`init: { flag }`](#relations) | a to-one relation defaults to the target row holding a `single` flag |
