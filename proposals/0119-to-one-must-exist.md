# A to-one naming a record that does not exist is refused on create and update

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/163, https://github.com/eclipse-dirigible/dirigible/issues/8100, https://github.com/eclipse-dirigible/dirigible/issues/7986
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8251

## The problem

1.11 states what deleting a referenced record does to the rows that point at it
(`whenTargetDeleted: restrict | nullify | cascade`), and `required` means NOT NULL - presence only.
Nothing states the other side of the reference: that a create or an update naming a to-one id must
name a record that exists. So a write naming an id with no row behind it was accepted and stored a
row pointing at nothing - same-model and cross-model alike, a self-reference (`Manager`) and a
composition master included (and an attachment upload under a missing master left its file behind).
A numbered document had already drawn its number and started its approval process by then.

## The proposed shape

No new key. The rule holds for every to-one a file already declares:

```yaml
- name: SalesOrder
  relations:
    - { name: Customer, kind: manyToOne, to: Customer, required: true }  # must name a Customer that exists
    - { name: Company,  kind: manyToOne, to: Company, model: companies } # cross-model: the same rule
- name: Employee
  relations:
    - { name: Manager,  kind: manyToOne, to: Employee }                  # a self-reference: the same rule
```

## Expected behaviour

- A create or an update naming a to-one target with no row behind it is refused with a 400 naming
  the relation and the id (`Customer 42 does not exist`), and nothing is stored.
- The rule holds same-model and cross-model, for a self-reference and for a composition master; an
  attachment upload checks its master before it stores the file.
- The check runs before a document number is drawn, so a refused create spends no number.
- An update is asked only about the references it changes, so a row stored before the rule existed
  stays editable.
- Inside one unit of work the check sees the unit's own uncommitted rows, so a header created earlier
  in the same create-from satisfies its lines.

## Edge rules

- A relation declaring `whenTargetDeleted: keep` is exempt: its reference outlives its target by
  declaration (a reversal names a source that is already gone). `keep` is not among the values
  1.11 lists for `whenTargetDeleted`; the exemption applies where a version of the format declares
  it.
- A target whose model is not deployed is let through, with a warning.
- The check is in the generated application, never a database constraint.
- Not covered: a targeted single-column write of a to-one key (a workflow setter writing it by name),
  and the race with a concurrent delete of the target - a read-then-act guard, like the delete side.

## Prior art / workarounds

None; the dangling row was stored. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8251. An editorial statement of a decided
behaviour, tracked in the 1.11 "states precisely" checklist.

## Specification text

**Anchor:** Relations & multi-model > relations > Relation attributes, appended after the
`whenTargetDeleted` paragraph and its normative block.

A to-one names a record that exists. A create or an update that names a target with no row behind it
is refused, whichever model owns the target.

> **Normative.**
> A conforming generator MUST refuse a create or an update that names a to-one target with no row
> behind it with a 400 naming the relation and the id, same-model and cross-model alike, a
> self-reference and a composition master included; an attachment upload MUST check its master
> before it stores the file. The check is in the generated application, never a database
> constraint. It MUST run before a document number is drawn, so a refused create spends no number.
> An update MUST be asked only about the references it changes, so a row stored earlier stays
> editable. A relation declaring `whenTargetDeleted: keep` is exempt, because its reference outlives
> its target by declaration; a target whose model is not deployed is let through with a warning.
