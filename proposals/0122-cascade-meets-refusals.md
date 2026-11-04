# A cascade meets the cascaded row's own refusals

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/163, https://github.com/eclipse-dirigible/dirigible/issues/8114
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8424

## The problem

`whenTargetDeleted: cascade` (0064) deletes the referencing rows "through their own delete path so
that their rules apply in turn", and `whenMasterDeleted: cascade` (0033) deletes each composition
child "in full". Neither says that the rules which **refuse** a row's delete - its period lock,
`immutableWhen` / `immutable`, a `forbidWhen` covering the delete - are among the rules that apply.
They were enforced on the generated user surfaces only, and no cascade passes through one: deleting
an employee removed a payout dated in a closed pay period, whose own delete answers 409, and
deleting a folder removed a note whose own delete a `forbidWhen` refuses with 400.

This is a clarification of what a conforming generator must refuse, not a new key.

## The proposed shape

No new key. The example that must hold:

```yaml
entities:
  - name: Payout
    immutableInPeriod: { period: PayPeriod, date: payDate }
    relations:
      - { name: employee, kind: manyToOne, to: Employee, whenTargetDeleted: cascade }
  - name: Note
    relations:
      - { name: folder, kind: manyToOne, to: Folder, composition: true }
    checks:
      - { kind: forbidWhen, when: "locked == true", message: "A locked note stays" }
```

Deleting an employee with a payout in a closed period is refused with the payout's own 409 and its
message, and the payout stays. Deleting a folder holding a locked note is refused with the note's
400 and its message, and the folder and all its notes stay.

## Expected behaviour

- A cascade asks each row it would delete the refusals that row's own delete meets - its period
  lock, `immutableWhen` / `immutable`, and a `forbidWhen` covering the delete - before anything is
  written: no nullify, no earlier cascade, no history row.
- The target's or master's delete is refused with the child's own message and status: 409 for a
  lock, 400 for a `forbidWhen`.
- The whole delete stays all-or-nothing.
- A child with nothing to refuse adds no check.
- Under `whenMasterDeleted`, the refusals are asked after the `refuse` compositions are decided and
  before the first child is deleted.

## Edge rules

- The lock a composition child inherits from its master and a process `whenDeleted: refuse` are the
  master's own concern and are not asked again per child.
- Each level of cascade asks its own direct rows. A refusal further down is met when that level's
  cascade runs, and the unit of work still takes back what the levels above it deleted.
- The refusals are asked by the cascade, whichever writer deleted the target. A system or workflow
  writer deleting the locked row itself directly is unaffected, as `immutableWhen` already states.

## Prior art / workarounds

None in the model: the cascade bypassed the refusals. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8424.

## Specification text

**Anchor:** Relations & multi-model > relations > Relation attributes, after the normative block
"Both outcomes MUST hold for every writer that can delete the master ...". States precisely the
`whenTargetDeleted: cascade` rule "so that their rules apply in turn" and the
`whenMasterDeleted: cascade` rule "a deletion in full".

A row deleted by a cascade is refused what its own delete is refused. The cascade asks first, so a
refused row leaves everything in place.

> **Normative.**
> A cascade MUST ask each row it would delete the refusals that row's own delete meets - its period
> lock, `immutableWhen` / `immutable`, and a `forbidWhen` covering the delete - before anything is
> written: no nullify, no earlier cascade, no history row. The master's delete is refused with the
> child's own message and status (409 for a lock, 400 for a `forbidWhen`), and the whole delete
> stays all-or-nothing. A child with nothing to refuse adds no check. The inherited master lock and
> a process `whenDeleted: refuse` are the master's own concern and are not asked again per child.

## DSL index

No new row; the `whenTargetDeleted` and `relations` / `whenMasterDeleted` rows are unchanged.
