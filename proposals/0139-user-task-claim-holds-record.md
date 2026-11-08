# A user task's claim holds its record

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8461
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8545

## The problem

A process started when a document is created reaches its first user task while the document is
still in the status it was created in. The approval of a sales order waits with the order in DRAFT,
so a `forbidWhen: "SalesOrder.Status != DRAFT"` on its lines freezes nothing while the approver
decides. The approver claims the task and reads the total; meanwhile the author changes a line's
quantity from 1 to 1000; the approver completes the task, and the order is confirmed at a total
nobody approved.

Nothing in 1.13 can move the record when its task is claimed. `immutableWhen` and `forbidWhen` read
a status, and nothing sets one at the claim. Splitting the step into a submit task and an approval
task moves the problem to the first task, and starting the flow on a transition needs a separate
submit act, which does not fit a document whose task must exist from creation.

## The proposed shape

```yaml
processes:
  - name: SalesOrderApproval
    trigger: { onCreate: SalesOrder }
    steps:
      - { name: confirm, kind: userTask,
          args: { assignee: approver, form: ConfirmSalesOrder,
                  onClaim:   { setRelationField: Status, value: SUBMITTED },
                  onUnclaim: { setRelationField: Status, value: DRAFT } } }
```

The lines then take `forbidWhen: "SalesOrder.Status != DRAFT"`, which holds for as long as the task
is claimed.

## Expected behaviour

- **Claim.** When the task is claimed, by whatever path, the trigger record's relation is set to the
  `onClaim` value. A task created already assigned to someone counts as claimed from its creation.
- **Unclaim.** When the task is unclaimed, the relation is set back to the `onUnclaim` value, and
  the record is the author's to change again.
- **Only from the other half's status.** A claim moves the record only from the `onUnclaim` value,
  and an unclaim only from the `onClaim` value. A task passing from one claimant to another does not
  move the record, and a record another writer has moved meanwhile is left as it is.
- **The lifecycle judges both moves.** A move the record's lifecycle refuses fails the claim (or the
  unclaim), and the user is told why.
- **A task that holds its record completes only claimed.** Completing such a task while nobody has
  claimed it is refused (a conflict), so no decision is taken over a record nothing held.
- The move is the same write a `setRelationField` step makes: it is a system write, observable as a
  status transition, and it does not re-fire update reactions.

## Edge rules

- `onClaim` and `onUnclaim` are user-task args and are declared together; one without the other is
  an authoring error.
- Each is `{ setRelationField, value }`: a to-one relation of the process's trigger record itself
  (not one hop away) and a seeded name or record id. Any other key is an authoring error.
- Both name the same relation, and their values differ.
- When the relation is the record's lifecycle status, the lifecycle MUST declare both edges -
  from the `onUnclaim` value to the `onClaim` value and back - or the file is refused.
- A task that leaves without completing (a boundary timer, a cancelled instance) does not move the
  record back; the model takes the record on from there itself.
- A file written for 1.13 declares neither arg and regenerates unchanged.

## Prior art / workarounds

A hand-written listener on the task's assignment, or a two-task flow that moves the problem to the
first task. Reference implementation: https://github.com/eclipse-dirigible/dirigible/pull/8545.

## Specification text

**Anchor:** Processes & forms > processes > Task assignment, after 0127's normative block.

A user task may hold its record while someone decides it: `onClaim` moves the trigger record into a
status when the task is claimed, and `onUnclaim` moves it back when the task is unclaimed.

```yaml
- { name: confirm, kind: userTask,
    args: { assignee: approver, form: ConfirmSalesOrder,
            onClaim:   { setRelationField: Status, value: SUBMITTED },
            onUnclaim: { setRelationField: Status, value: DRAFT } } }
```

> **Normative.**
> A user task declaring `onClaim` and `onUnclaim` MUST set the named to-one relation of the
> process's trigger record to the `onClaim` value when the task is claimed, and to the `onUnclaim`
> value when it is unclaimed, through every path that claims or unclaims it; a task created with an
> assignee counts as claimed. Each move MUST happen only from the other value, and MUST be subject
> to the record's lifecycle, a refused move failing the claim or unclaim. Such a task MUST NOT be
> completable while it is unclaimed. Declaring only one of the two, a key other than
> `setRelationField` and `value`, a relation that is not a to-one relation of the trigger record, two
> different relations, or the same value twice MUST be reported as authoring errors; on the
> lifecycle status, a missing edge between the two values MUST be reported as an authoring error.

## DSL index

| Construct | What it does |
| --- | --- |
| [`onClaim`](#task-assignment) | on a user task: `{ setRelationField, value }` - the status the trigger record takes while the task is claimed |
| [`onUnclaim`](#task-assignment) | on a user task: `{ setRelationField, value }` - the status the record returns to when the task is unclaimed |
