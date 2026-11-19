# An approval refuses its submitter

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/163, https://github.com/eclipse-dirigible/dirigible/issues/8111
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8437

## The problem

Task assignment routes a user task to a candidate group, to the record's owner, or along a relation
walk, and says nothing about who must **not** complete it. A deployment that offers every task to
a group every user holds - so that its users, who hold no role of the model, can take any task at
all - makes every user a candidate for every task, the approval included. The requester submits an
expense claim, a leave request or a timesheet, then claims the approval and approves it.

Separation of duties is a property of the process, not of the deployment's groups. This states the
default a conforming generator must enforce; it adds no key.

## The proposed shape

No new key. The example that must hold:

```yaml
processes:
  - name: leaveApproval
    trigger: { onCreate: LeaveRequest }
    steps:
      - { name: submit,  kind: userTask, args: { assignee: personal, form: SubmitLeave } }
      - { name: approve, kind: userTask, args: { assignee: approver, form: ApproveLeave } }
forms:
  - { name: ApproveLeave, forEntity: LeaveRequest, fields: [fromDate, toDate],
      actions: [approve, reject] }
```

The user who completed `submit` is refused when claiming or completing `approve`, with a message
such as "You cannot approve your own leave request"; a colleague holding the group approves it.

## Expected behaviour

- A user task whose form offers the `approve` action is separated from every other user task of its
  process: the user who completed one of those tasks may neither claim nor complete it.
- The refusal is a 403 where the surface is HTTP, carrying a message that names what the task
  approves.
- No candidate group lifts the refusal, including one that every user of the deployment holds.
- The completer is recorded by the platform when the task is completed, not taken from the
  completing request, so a caller cannot plant it.

## Edge rules

- The rule refuses claim and completion; it does not by itself remove the task from the
  submitter's inbox.
- The rule binds assigned tasks too: a relation walk that routes the approval back to the user who
  submitted - an employee recorded as their own manager - reaches a task that user is refused, and
  the task has to be reassigned.
- A model generated before the rule has no separated tasks and behaves as before until it is
  regenerated.

## Prior art / workarounds

None in the model: a hand-written check on the task's completion. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8437.

## Specification text

**Anchor:** Processes & forms > processes > Task assignment, appended.

Who may take a task is one half of assignment; who may not is the other. An approval is not taken
by the user who submitted what it approves.

> **Normative.**
> A user task whose form offers the `approve` action MUST NOT be claimable, nor completable, by the
> user who completed another user task of the same process instance - the user who submitted the
> record it approves. The refusal MUST be a 403 where the surface is HTTP, carrying a message that
> names what the task approves, and a candidate group that every user of the deployment holds MUST
> NOT widen it. The completer of a task MUST be recorded by the implementation at
> completion, never taken from the completing request.

## DSL index

No new row; the `processes` and `forms` rows are unchanged.
