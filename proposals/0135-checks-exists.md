# checks: kind: exists - a write refused unless a matching row of another entity exists

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8338
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8529

## The problem

Every check kind reads the record being written and, at most, the records its own to-one relations
point at. None can say "a row of a THIRD entity must exist for the values this record names". The
common case: a billable expense line names a project, its claim names an employee, and the line may
be billed to that project only when an assignment row links the two. `agree` compares a property
both linked targets declare, but the assignment is neither target - it is a third entity (here a
composition child of the project, owned by another model). `forbidWhen` / `requiredWhen` read the
record and one hop; they cannot test whether a row exists. So the rule was either hand-written as a
calculation that throws, or missing, and the line was stored. The same need appears wherever a
booking is restricted to members: a timesheet entry on a project, an approver from the requester's
team, a cashier on a register.

## The proposed shape

```yaml
- name: ExpenseClaimItem
  checks:
    - { kind: exists, entity: EmployeeProjectAssignment, model: projects,
        match: { Project: Project, Employee: ExpenseClaim.Employee },
        when: "billable == true", verbs: [create, update],
        message: "Only an employee assigned to the project bills an expense to it" }
- name: ExpenseClaim
  checks:
    - { kind: exists, entity: EmployeeProjectAssignment, model: projects,
        match: { Project: Project, Employee: Employee }, status: SUBMITTED,
        message: "Only an employee assigned to the project submits a claim for it" }
```

- `entity` - the entity a matching row must exist in. Required.
- `model` - optional: the `uses:` model that owns `entity`; omitted for an entity of this file.
- `match` - required, non-empty: `<property of entity>: <path on this record>`. The path is the
  record's own field or to-one, or a one-hop `Relation.field` over a to-one - the path vocabulary
  `requiredWhen`'s `field` uses. Every pair must hold on ONE row.
- `when` - optional: the condition under which the rule applies, in the `forbidWhen` grammar.
- `verbs` - optional, and only `[create, update]`: the check is about the values a write carries.
- `status` - optional: the gate, with the routing `requiredWhen` and `compare` have.
- `message` - required: the refusal.

## Expected behaviour

- A create or update whose `when` holds and whose match values are all present is refused with
  `message` unless a row of `entity` carries all of them.
- A match value that is absent skips the check - whether the value must be present is `required`'s
  declaration, as with `agree`'s default for an unset side.
- Without `status` the rule holds on every user write; with it, when the record is persisted carrying
  that status, and the refusal reaches whoever performed the transition.
- The message is translated through the module's catalogs like every other check message.

## Edge rules

- A cross-model `entity` resolves through its `uses:` owner. A `model` that `uses:` does not declare
  is an authoring error; so is a `match` property the target does not declare.
- `entity` names another entity: a row of the record's own entity always matches the row an update
  writes.
- A pair whose two sides are of different kinds (a reference against a field, two field types) is an
  authoring error - no row would ever match.
- Removing the matching row later does not revisit the records already stored; that is the target's
  delete rule's business.

## Prior art / workarounds

A calculation action that throws - a calculation, not a refusal, firing only on the field that
declares it. Reference implementation: https://github.com/eclipse-dirigible/dirigible/pull/8529.

## Specification text

**Anchor:** Entities & fields > checks - declarative validations, a new `#### kind: exists`
section after `#### kind: agree — two relations that must agree on a shared target`.

#### kind: exists — a write that needs a matching row of another entity

An `exists` check names another entity and the values a row of it must carry, read off the record
being written. It refuses the write while no such row exists:

```yaml
- name: ExpenseClaimItem
  checks:
    - { kind: exists, entity: EmployeeProjectAssignment, model: projects,
        match: { Project: Project, Employee: ExpenseClaim.Employee }, when: "billable == true",
        message: "Only an employee assigned to the project bills an expense to it" }
```

- `entity` — the entity the matching row belongs to; with `model`, an entity of the `uses:` model
  of that name.
- `match` — each property of `entity` and the path on the record its value must equal: a field or
  to-one of the record, or a one-hop `Relation.field`. All pairs hold on one row.
- `when` — optional; the condition under which the rule applies, in the `forbidWhen` grammar.
- `verbs` — optional; only `[create, update]`.
- `status` — optional; without it the check holds on every user write, with it when the record is
  persisted carrying that status.
- `message` — the authored text the refused write is reported with.

> **Normative.**
> A conforming generator MUST refuse a create or update of the record, with `message`, when `when`
> holds (or is absent), every `match` value is present, and no row of `entity` carries all of them;
> a `match` value that is absent MUST NOT refuse the write. With `status`, it MUST evaluate the check
> when the record is persisted carrying that status instead, and the gate requires a
> `function: EntityStatus` relation. The lookup MUST pass the values as bound parameters, never as
> query text. Each of the following MUST be reported as an authoring error: a missing `entity`,
> `match` or `message`; a `model` the file's `uses:` does not declare; an `entity` that is not
> declared, or is the record's own entity; a `match` path that does not resolve; a `match`
> property the target does not declare; a pair whose two sides are of different kinds; `verbs`
> naming `delete`; and `severity: warn`.

## DSL index

A new row, after `checks: kind: agree`:

| Construct | What it does |
| --- | --- |
| [`checks: kind: exists`](#kind-exists--a-write-that-needs-a-matching-row-of-another-entity) | a write refused unless a row of another entity - of this file or a `uses:` model - carries the values `match` reads off the record (own or one hop); a null value skips it; on every write or at a status |
