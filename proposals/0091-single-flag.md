# `single` - a boolean held by at most one row, fleet-wide or per a to-one scope

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/152, https://github.com/eclipse-dirigible/dirigible/issues/8101
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8248

## The problem

Every nomenclature carries a "default" boolean: the base currency, the default unit, tax rate,
payment method or company, one default bank account per company. No construct of 1.11 keeps it
single. `unique: true` on a boolean admits two rows, one per value; an entity-level `unique` over
`[company, isDefault]` refuses a second `false` as readily as a second `true`; and `checks:` cannot
count sibling rows. So a second default is stored and answered as a success, and every
reader that wants "the" default gets one of two.

## The proposed shape

```yaml
fields:
  - { name: base,      type: boolean, single: true }                                   # one row of the whole entity
  - { name: isDefault, type: boolean, single: { per: company, onConflict: demote } }   # one per company; the new one wins
  - { name: primary,   type: boolean, single: { per: company, onConflict: refuse } }   # one per company; the second is refused
```

## Expected behaviour

- A row written with the flag `true` looks up the other holders of the flag in its scope - every
  row of the entity (`single: true`), or the rows sharing the value of the `per:` to-one - once every
  other guard of the write has passed, and in the same unit of work as the write.
- `onConflict: demote` (the default) writes each other holder `false` through that holder's own
  update, so its checks, its history and its update event run; the demotion commits or rolls back
  with the write that caused it.
- `onConflict: refuse` answers the write with a conflict (409) and a translatable message, and writes
  nothing.
- A row written with the flag `false` is not concerned. The form keeps the usual checkbox.

## Edge rules

- Refused when the intent is read: `single` on a field that is not `boolean`; a `per` that is not a
  `manyToOne` / `oneToOne` relation of the same entity; an unknown `onConflict`; a value that is
  neither a boolean nor a mapping. `single: false` declares nothing.
- Not covered: a targeted single-column write of the flag, which runs no full-row write; and two
  concurrent writers that each find no holder.
- A model without `single` generates byte-identically.

## Prior art / workarounds

None: each module stored a second (or third) default, answered 200. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8248.

## Specification text

**Anchor:** Entities & fields > fields. A row in the attribute table, after `unique`; and a
subsection after "unique - a business key over more than one field", before "renamedFrom / dropped".

| `single` | on a `boolean`: at most one row holds the value `true`, among every row or per a to-one relation's value; a second holder is demoted or refused (see [single](#single---at-most-one-row-holds-the-flag)) |

#### single - at most one row holds the flag

A `unique` boolean admits two rows, one per value. Where a boolean marks the one row that is the
default - the base currency, the default bank account of a company - declare it `single`:

```yaml
fields:
  - { name: base,      type: boolean, single: true }
  - { name: isDefault, type: boolean, single: { per: company, onConflict: demote } }
  - { name: primary,   type: boolean, single: { per: company, onConflict: refuse } }
```

`single: true` keeps the flag on one row of the entity. `per` names a to-one relation of the same
entity whose value scopes it: one holder per company. `onConflict` decides what a write that sets
the flag does to the row already holding it: `demote` (the default) takes it away from that row,
`refuse` refuses the write.

> **Normative.**
> When a write stores a row with a `single` flag `true`, a conforming generator MUST find the other
> rows holding the flag in the row's scope - every row of the entity, or the rows whose `per`
> relation names the same target - after every other check and guard of the write has passed and in
> the same unit of work as the write. Under `onConflict: demote` it MUST write each of them `false`
> through that row's own update, so the row's checks, history and update event apply, and the
> demotions MUST commit or roll back with the write that caused them. Under `onConflict: refuse` it
> MUST refuse the write as a conflict a caller can distinguish from a generic failure, with a message
> seeded in the translation catalog, and MUST store nothing. A row whose flag is `false` is not
> concerned. A generator MUST reject as an authoring error: `single` on a field that is not
> `boolean`; a `per` that is not a `manyToOne` or `oneToOne` relation of the same entity; an
> `onConflict` other than `demote` and `refuse`; and a value that is neither a boolean nor a mapping.
> An implementation is NOT required to apply the rule to a targeted write of the flag alone, nor to
> serialise two concurrent writes that each find no holder. A model declaring no `single` MUST
> generate byte-identical output.

## DSL index

| Construct | What it does |
| --- | --- |
| [`single`](#single---at-most-one-row-holds-the-flag) | a boolean that at most one row holds, among every row or per a to-one scope; the previous holder is demoted (default) or the write refused with 409 |
