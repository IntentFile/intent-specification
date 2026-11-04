# A member fixed while other records reference the row - `fixedWhenReferenced`

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/151, https://github.com/eclipse-dirigible/dirigible/issues/8194
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8429

## The problem

The delete of a referenced row is refused ([`whenTargetDeleted: restrict`](0064-when-target-deleted.md)),
but a plain update may rewrite what every referencing record means: a currency's code, a bank
account's IBAN or currency, a product's type. Once a payment in EUR exists, renaming the currency's
code to USD changes the payment without touching it.

Nothing in 1.11 guards that. `immutableWhen` is scoped to the row's own status; a check cannot see
other models' rows; [`fixedAfterCreate`](0089-fixed-after-create.md) fixes the value outright, before
anything uses it; and the referencing module cannot guard the referenced module's update, because
it is generated after it and the referenced model knows nothing about it.

## The proposed shape

```yaml
entities:
  - name: Currency
    fields:
      - { name: code, type: string, fixedWhenReferenced: true }
  - name: CompanyBankAccount
    relations:
      - { name: Currency, kind: manyToOne, to: Currency, model: currencies, fixedWhenReferenced: true }
```

## Expected behaviour

- A user update that changes such a field, or such a to-one's key, on a row that **other records
  reference** is answered **409** with a translatable message, by default
  `'Code' cannot change while other records reference this Currency`, and stores nothing.
- Whether the row is referenced is asked only when the payload changes the member. It is answered by
  every entity that references the row through a to-one carrying a `whenTargetDeleted` rule -
  `restrict` (declared or defaulted), `nullify` or `cascade` - same-model and cross-model. A
  composition child's reference to its master does not count.
- Values are compared as stored: a value the application normalises before storing after that
  normalisation, a decimal by value (`150.00` against a stored `150` is no change), a relation by
  its key. An omitted value keeps the stored one; repeating the stored value saves.
- An unreferenced row stays editable, and becomes editable again once its last reference is gone.
- The rule holds on the regular, personal and partner surfaces alike.
- System writes - roll-ups, postings, process steps, migrations - are unaffected.

## Edge rules

- Refused when the intent is read: on the primary key, on a calculated field (`calculatedOnCreate`,
  `calculatedOnUpdate`, `calculatedActionOnUpdate`), beside `fixedAfterCreate` (which already fixes
  the value outright), and everywhere `fixedAfterCreate` is refused on a field.
- On a relation it is accepted on a `manyToOne` / `oneToOne` only, and refused on a `oneToMany`, a
  `subset`, a `manyToMany`, a composition parent (already fixed, never re-picked), a
  `function: EntityStatus` relation (the workflow moves it), and beside `calculatedActionOnUpdate`
  (that recompute is a system write).
- The edit form does not lock the input: only the server knows whether the row is referenced.
- A reference inserted between the check and the write is not seen - the same limit a
  [`kind: agree`](../versions/1.11.md#kind-agree--two-relations-that-must-agree-on-a-shared-target)
  check has.

## Prior art / workarounds

A hand-written guard in the referenced module's custom code that enumerates its referrers, which it
cannot know when they live in models generated after it. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8429.

## Specification text

**Anchor:** Entities & fields > fields, a row in the attribute table after `fixedAfterCreate`
([0089](0089-fixed-after-create.md)); a new section after the `fixedAfterCreate` section (after
Entities & fields > defaultValue - field defaults, if 0089 is not folded); and Relations &
multi-model > Relation attributes, a bullet after `whenTargetDeleted`, whose update side it is.

Row in the fields table:

| Key | Meaning |
| --- | --- |
| `fixedWhenReferenced` | the value cannot be changed by a user update while other records reference the row (see [fixedWhenReferenced](#fixedwhenreferenced---fixed-while-the-row-is-referenced)) |

New section:

### fixedWhenReferenced - fixed while the row is referenced

What a referenced row says is part of what every record pointing at it means: the code of a currency
a payment was made in, the IBAN of an account a transfer went to. `whenTargetDeleted` keeps such a
row from disappearing; `fixedWhenReferenced: true` keeps the member from changing under its
referrers, while leaving it editable as long as nothing references the row:

```yaml
- name: Currency
  fields:
    - { name: code, type: string, fixedWhenReferenced: true }
- name: CompanyBankAccount
  relations:
    - { name: Currency, kind: manyToOne, to: Currency, model: currencies, fixedWhenReferenced: true }
```

The referenced model does not know its referrers; the relations that reference it say so, through
their `whenTargetDeleted` rule, and the same rule that answers a delete answers this update.

> **Normative.**
> A user update that changes a `fixedWhenReferenced` member of a row that other records reference
> MUST be refused with a 409 and a translatable message naming the member and the entity (by default
> `'<Member>' cannot change while other records reference this <Entity>`), and MUST store nothing.
> A row is referenced when a row of any entity, of this model or another, points at it through a
> to-one whose `whenTargetDeleted` rule is `restrict` (declared or defaulted), `nullify` or
> `cascade`; a composition child's reference to its master does not count. The question MUST be
> asked only when the update changes the member, and values MUST be compared as stored: a decimal by
> value, a value the application normalises before storing after that normalisation, a to-one by
> its key. An update that leaves the member out MUST keep the stored value, one that repeats it MUST
> save, and an unreferenced row MUST stay editable. The rule MUST hold on every surface a user writes
> the record through and governs user writes only: roll-ups, postings, process steps and migrations
> MAY write the member. The edit form is not required to lock the input, since whether the row is
> referenced is known only when the update is made.

> **Normative.**
> `fixedWhenReferenced` MUST be reported as an authoring error on a field wherever `fixedAfterCreate`
> is - the primary key and the calculated fields included - and beside `fixedAfterCreate`, which
> already fixes the value outright. On a relation it MUST be accepted on a `manyToOne` or `oneToOne`
> only, and MUST be reported as an authoring error on a `oneToMany`, a `subset`, a `manyToMany`, a
> composition parent, a `function: EntityStatus` relation, and beside `calculatedActionOnUpdate`. A
> model that declares no `fixedWhenReferenced` MUST generate as it did before.

The check reads the referrers before the write, so a reference added between the two is not seen,
as with a `kind: agree` check.

New bullet in Relations & multi-model > Relation attributes, after `whenTargetDeleted`:

- **`fixedWhenReferenced: true`** on a plain `manyToOne` / `oneToOne`: the reference cannot be
  re-picked by a user update while other records reference this row, with the rules of
  [fixedWhenReferenced](#fixedwhenreferenced---fixed-while-the-row-is-referenced) - the update-side
  counterpart of `whenTargetDeleted`.

## DSL index

| Construct | What it gives you |
| --- | --- |
| [`fixedWhenReferenced`](#fixedwhenreferenced---fixed-while-the-row-is-referenced) | a field or plain to-one refused (409) to a user update that changes it while records of any model reference the row through a `whenTargetDeleted` rule; editable while unreferenced |
