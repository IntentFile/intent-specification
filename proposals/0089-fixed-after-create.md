# A value picked on create and fixed afterwards - `fixedAfterCreate`; a composition child's master is fixed by default

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/150, https://github.com/eclipse-dirigible/dirigible/issues/7903, https://github.com/eclipse-dirigible/dirigible/issues/8363, https://github.com/eclipse-dirigible/dirigible/issues/8306
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8246, https://github.com/eclipse-dirigible/dirigible/pull/8378, https://github.com/eclipse-dirigible/dirigible/pull/8425

## The problem

Some values are the user's choice, made once: the employee a timesheet belongs to, the
project-month it books against, a document's reference, a rate agreed when the record was made.
Once the record exists they must not move - otherwise an employee can move their own timesheet
onto another customer's invoice. No key in 1.11 says so:

- `readOnly` is "rendered read-only in the UI", so the create form cannot take the value either,
  and a relation has no `readOnly` at all.
- A derived "keep the stored value" action (`calculatedActionOnUpdate`) is read as a derived value,
  so the picker is disabled on the create form too.
- `immutableWhen` locks the whole record by status, while a DRAFT timesheet must stay editable.

Separately, a `composition: true` child's reference to its master is writable on update in 1.11. An
update of a line that names another master moves the line, and its amounts, to that master, and both
masters' totals follow.

## The proposed shape

```yaml
entities:
  - name: Timesheet
    fields:
      - { name: reference, type: string,  fixedAfterCreate: true }
      - { name: rate,      type: decimal, fixedAfterCreate: true }
    relations:
      - { name: employee, kind: manyToOne, to: Employee, fixedAfterCreate: true }
  - name: InvoiceItem
    relations:
      - { name: invoice, kind: manyToOne, to: Invoice, composition: true }   # fixed by default
```

## Expected behaviour

- The create form takes the value; the edit form shows it read-only (the picker of a relation is
  disabled and offers no "new" affordance).
- A user update that carries a **different** value than the stored one is answered **409** with a
  translatable message, by default `'<Member>' cannot change once the <Entity> is created`, and
  stores nothing. An update that leaves the value out keeps the stored one; one that repeats it
  saves.
- Values are compared as stored: a decimal by value (`100` against a stored `100.00` is no change),
  a value the application normalises before storing after that normalisation (`sys-1` against a
  stored `SYS-1` under upper-case normalisation is no change), a relation by its key.
- The rule holds on the regular, personal and partner surfaces alike.
- System writers - a process step that sets the field or relation, a create-from, a roll-up,
  hand-written code - are unaffected: the rule governs user writes.
- **A composition child's master reference is fixed after create without any key.** An update
  naming another master is answered 409 with its own translatable message, by default
  `This <Child> belongs to its <Master> - to move it, delete it and add it to the other <Master>`;
  leaving the master out or repeating it saves. On a personal or partner surface a different parent,
  formerly replaced silently by the stored one, is now refused the same way. The edit form renders
  the master read-only.

## Edge rules

- On a relation the key is accepted on a `manyToOne` / `oneToOne` only. It is refused on a
  `function: EntityStatus` relation (the workflow moves it), a `oneToMany`, a `subset`, a
  `manyToMany`, and beside `calculatedActionOnUpdate`. On a composition it is accepted and changes
  nothing.
- On a field it is refused on the primary key, a `generated` or `uuid` field, a `number:` field
  (whatever its `stampOn`), `readOnly: true`, a calculated field (`calculatedOnCreate`,
  `calculatedOnUpdate`, `calculatedActionOnUpdate`), an `aggregate` field, a `dependsOn` field, a
  `timestamp`, and a translatable string of a `multilingual` entity (the message says to add
  `translatable: false`). Every refusal names the entity, the field and the reason.
- A `timestamp` is refused because an edit form shows it to the minute and re-sends a different
  instant, so an unchanged save would be refused. A `date` round-trips exactly and is accepted.
- It is accepted beside `calculatedActionOnCreate`, `defaultValue`, `unique`, `pattern` and
  `format`: a create-time default may be fixed.
- A model that declares no `fixedAfterCreate` generates as before, except that a composition child's
  master is now fixed.
- Line editors of a document's items are not required to lock the member in their inline editing;
  the child's own update is still refused.

## Prior art / workarounds

A hand-written guard per entity in the project's custom code (the timesheet module carried one), or
a `calculatedActionOnUpdate` that copies the stored value back, which also disabled the create
form's input. For the composition master, nothing: a line could be re-parented by any client that
sent another master's key. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8246 (to-one),
https://github.com/eclipse-dirigible/dirigible/pull/8378 (field),
https://github.com/eclipse-dirigible/dirigible/pull/8425 (composition master).

## Specification text

**Anchor:** Entities & fields > fields, a row in the attribute table after `readOnly`; a new section
after Entities & fields > defaultValue - field defaults; Relations & multi-model > relations, a
sentence and a normative block appended to the `composition: true` bullet; and Relations &
multi-model > Relation attributes, a bullet after `whenTargetDeleted`.

Row in the fields table:

| Key | Meaning |
| --- | --- |
| `fixedAfterCreate` | the value is chosen on create and cannot be changed by a user update afterwards (see [fixedAfterCreate](#fixedaftercreate---set-on-create-fixed-afterwards)) |

New section:

### fixedAfterCreate - set on create, fixed afterwards

Some values are the user's choice, made once: the employee a timesheet belongs to, a document's
reference, a rate agreed when the record was made. They are neither derived nor seeded, so
`readOnly` does not fit, and the rest of the record stays editable, so a status lock does not fit
either. `fixedAfterCreate: true` declares such a value, on a field or on a plain to-one:

```yaml
- name: Timesheet
  fields:
    - { name: reference, type: string,  fixedAfterCreate: true }
    - { name: rate,      type: decimal, fixedAfterCreate: true }
  relations:
    - { name: employee, kind: manyToOne, to: Employee, fixedAfterCreate: true }
```

The create form takes the value like any other; the edit form shows it read-only. An update that
leaves the value out, or repeats it, saves; one that changes it is refused.

> **Normative.**
> A conforming generator MUST accept a `fixedAfterCreate` member's value on create and MUST render
> it read-only on the record's edit form. A user update that carries a value different from the
> stored one MUST be refused with a 409 and a translatable message naming the member and the entity
> (by default `'<Member>' cannot change once the <Entity> is created`), and MUST store nothing. An
> update that leaves the member out MUST keep the stored value, and one that repeats it MUST save.
> Values MUST be compared as stored: a decimal by value, a value the application normalises before
> storing after that normalisation, a to-one by its key. The rule MUST hold on every surface a user
> writes the record through, the personal and partner surfaces included. It governs user writes
> only: a process step, a create-from, a roll-up and hand-written code MAY write the member.

> **Normative.**
> On a field, `fixedAfterCreate` MUST be reported as an authoring error on the primary key, a
> `generated` or `uuid` field, a `number:` field, a `readOnly: true` field, a calculated field
> (`calculatedOnCreate`, `calculatedOnUpdate`, `calculatedActionOnUpdate`), an `aggregate` field, a
> `dependsOn` field, a `timestamp`, and a translatable string of a `multilingual` entity, the last
> with a message directing the author to `translatable: false`. It MUST be accepted beside
> `calculatedActionOnCreate`, `defaultValue`, `unique`, `pattern` and `format`. On a relation it
> MUST be accepted on a `manyToOne` or `oneToOne` only, and MUST be reported as an authoring error
> on a `function: EntityStatus` relation, a `oneToMany`, a `subset`, a `manyToMany`, and beside
> `calculatedActionOnUpdate`. A model that declares no `fixedAfterCreate` MUST generate as it did
> before.

A `timestamp` is excluded because an edit form shows it to the minute and sends back a different
instant, so an unchanged save would be refused; a `date` round-trips exactly.

Appended to the `composition: true` bullet of Relations & multi-model > relations:

A composition child's reference to its master is fixed once the child exists: a line moves to
another document by being deleted and added there, never by an update.

> **Normative.**
> A user update of a composition child that names a master other than the stored one MUST be
> refused with a 409 and a translatable message naming the child and the master (by default
> `This <Child> belongs to its <Master> - to move it, delete it and add it to the other <Master>`),
> and MUST store nothing; an update that leaves the master out or repeats it MUST save. The rule
> needs no key and MUST hold on every surface, the personal and partner surfaces included, where a
> different parent MUST be refused rather than replaced by the stored one. The edit form MUST render
> the master read-only. System writers are not governed by it, as for `fixedAfterCreate`.

New bullet in Relations & multi-model > Relation attributes, after `whenTargetDeleted`:

- **`fixedAfterCreate: true`** on a plain `manyToOne` / `oneToOne`: the target is picked on create
  and fixed afterwards, with the rules of
  [fixedAfterCreate](#fixedaftercreate---set-on-create-fixed-afterwards). A composition's master is
  fixed already, so the key is accepted there and changes nothing.

## DSL index

| Construct | What it gives you |
| --- | --- |
| [`fixedAfterCreate`](#fixedaftercreate---set-on-create-fixed-afterwards) | a field or plain to-one chosen on create and refused (409) to a user update that changes it; a composition child's master is fixed the same way without the key |
