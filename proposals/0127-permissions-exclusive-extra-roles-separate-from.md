# What a deployment's shared roles may not reach, and whom an approval is separated from

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/218, https://github.com/eclipse-dirigible/dirigible/issues/8111, https://github.com/eclipse-dirigible/dirigible/issues/8112
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8437
- **Builds on:** [0121 - an approval refuses its submitter](0121-approval-refuses-submitter.md)

## The problem

A shared deployment adds roles that every one of its users holds to the gates the generated
application enforces, so that a user who holds no role of the model can open the application at
all. That reaches too far in two places. An entity holding salaries or bank details becomes
readable and writable by every user: a plain user changes a colleague's bank account. And every user
task becomes claimable by every user, so nothing keeps a task meant for a manager away from the
people it concerns.

0121 states the default separation of duties: an approval is not claimed or completed by the user
who completed another user task of its process. A model also needs to say which steps an approval
is separated from, or that it is not separated at all. 1.11 knows none of the three keys.

## The proposed shape

```yaml
permissions:
  - { role: HR, can: [Employee:write], exclusive: true }   # shared roles do not reach this grant

processes:
  - name: approval
    trigger: { onCreate: ExpenseClaim }
    steps:
      - { name: submit,  kind: userTask,
          args: { assignee: personal, form: SubmitClaim, extraRoles: true } }  # shared roles too
      - { name: review,  kind: userTask, args: { assignee: Manager, form: ReviewClaim } }
      - { name: approve, kind: userTask,
          args: { assignee: Finance, form: ApproveClaim, separateFrom: [submit] } }
```

## Expected behaviour

- **`exclusive: true`.** Every resource the entry binds - an entity, a report, an export - is gated
  by the model's own roles only, for reads and writes alike: the deployment's shared roles are not
  added to those gates. An exclusive entity refuses the shared roles on read and on write. A
  composition child with no grants of its own follows its master, as it does for any authored gate.
  Every surface that reads the entity's gate honours it, the create-from and transition gates and
  the print feed included.
- **`extraRoles: true`.** A user task is offered to its `assignee` only; the deployment's shared
  roles are not among its candidates. A step declaring `extraRoles: true` offers the task to them
  as well.
- **`separateFrom`.** By 0121's default, a user task whose form offers `approve` is separated from
  every other user task of its process. `separateFrom: <step>` or `separateFrom: [a, b]` names the
  steps it is separated from instead; `separateFrom: []` turns the separation off. The refusal is
  0121's: a claim or a completion by a user who completed one of the named tasks is a 403 with a
  message naming what the task approves, for example "You cannot approve your own expense claim".

## Edge rules

- `separateFrom` naming a step that is not another user task of the same process is an authoring
  error. `extraRoles` that is not a boolean is an authoring error.
- A role-literal `assignee` that no `permissions` entry declares is reported at generation: no user
  of a shared deployment holds it, so only its declared holders could claim the task. The report
  points to `assignee: personal` or a relation walk.
- `exclusive` is declared where the gate is authored, on the `permissions` entry, so it covers
  reports and exports, which have no entity block.
- A file written for 1.11 regenerates with the default separation on its approval tasks and with
  its user tasks no longer offered to the shared roles; everything else is unchanged.

## Prior art / workarounds

None in the model: the shared roles reached every gate and every task, and the separation was a
hand-written check on the task's completion. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8437.

## Specification text

**Anchor:** Scoped surfaces & roles > permissions, after the `can:` paragraph, for `exclusive`;
Processes & forms > processes > Task assignment, after 0121's normative block, for `extraRoles`
and `separateFrom`.

*For Scoped surfaces & roles > permissions:*

A deployment may add roles every one of its users holds to every gate, so that those users reach
the application at all. A grant that must not be widened that way is declared `exclusive`:

```yaml
permissions:
  - { role: HR, can: [Employee:write], exclusive: true }
```

> **Normative.**
> Every resource a `permissions` entry declaring `exclusive: true` binds - an entity, a report, an
> export - MUST be gated, for reads and for writes, by the roles the model declares only; roles a
> deployment adds to every gate MUST NOT reach it. A composition child with no grants of its own
> follows its master's gate. Every generated surface that reads the resource's gate MUST honour the
> exclusion.

*For Processes & forms > processes > Task assignment:*

A user task is offered to its `assignee`. Roles a deployment adds to every gate are not among its
candidates unless the step declares `extraRoles: true`. An approval's separation from the other
user tasks of its process is the default; `separateFrom` names the steps instead, or turns it off:

```yaml
- { name: approve, kind: userTask,
    args: { assignee: Finance, form: ApproveClaim, separateFrom: [submit] } }
```

> **Normative.**
> A user task MUST NOT be offered to the roles a deployment adds to every gate unless its step
> declares `extraRoles: true`. A user task declaring `separateFrom: <step>` or
> `separateFrom: [a, b]` MUST NOT be claimable, nor completable, by the user who completed one of
> the named tasks in the same process instance, with the refusal stated for the default
> separation; `separateFrom: []` MUST turn the separation off. `separateFrom` naming a step that is
> not another user task of the same process, and `extraRoles` that is not a boolean, MUST be
> reported as authoring errors. A role-literal `assignee` that no `permissions` entry declares MUST
> be reported at generation.

## DSL index

| Construct | What it does |
| --- | --- |
| [`permissions[].exclusive`](#permissions) | `true` keeps the roles a deployment adds to every gate off the resources the entry binds, for reads and writes |
| [`extraRoles`](#task-assignment) | on a user task: `true` offers the task to the roles a deployment adds to every gate, which by default it is not |
| [`separateFrom`](#task-assignment) | on a user task: the steps whose completer may not claim or complete it - by default every other user task when its form offers `approve`; `[]` turns it off |
