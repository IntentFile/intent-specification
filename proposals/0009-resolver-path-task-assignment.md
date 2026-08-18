# Resolver-path user-task assignment


- **Status:** released in [1.3](../versions/1.3.md)
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/6716
- **Discussion:** https://github.com/IntentFile/intent-specification/pull/18

## The problem

A user task's `assignee` is a role / candidate-group name or the literal `personal` (the record owner). The reviewer is very often a person the **record itself names** — the requester's manager, the customer's account manager, the department's approver — and that has no expression today, even though the notify block already resolves one-hop paths off a record with the same kind of validation. The Planned list has carried it since 1.1.

## The shape

```yaml
- name: approve
  kind: userTask
  args:
    assignee: { path: employee.manager, fallback: manager }
    form: ApproveRequest
```

## The behaviour

- Every segment of `path` is a **to-one relation** — the first of the trigger entity, each further one of the previous target — and the walk ends at an entity that declares `identity`, which is what maps a record to a login.
- A **cross-model** relation may only be the **last** segment: a projection carries the target's own properties (so its identity is known) but not its relations, so there is nothing to walk on from there.
- A conforming generator **validates every hop when the file is read**, so a dangling segment is reported then rather than when the process runs.
- `fallback` is **required** and names the candidate group. The walk is resolved **when the task is reached**, not when the process starts — so a relation an earlier step of the same process set is visible — and when it resolves to nobody (a null hop, a missing record, a blank identity) the task is created **unassigned** and the fallback group can still claim it. That is what makes the unresolvable case total: a resolver path can never mint a task nobody can see.

## Prior art

Hand-written code: a delegate that loads the record, chases the foreign keys and calls the task service to set an assignee — the shape every workflow project rebuilds, with the null-hop case usually left to leave an unclaimable task.

## Implementation

Eclipse Dirigible: eclipse-dirigible/dirigible#6738 (closes eclipse-dirigible/dirigible#6716) — parse-time hop validation, a delegate inserted before the task, and the assignee expression plus fallback candidate group on the user task.

## Changes here

- `versions/1.2.md` — the *Task assignment* section, the scoped-surfaces cross-reference, a DSL index row, and the Planned entry removed.

## Specification text

The prose below is what a release folds into the next version document, at the anchors
given. It was written against `versions/1.2.md` and is carried here unchanged.

**Anchor:** Processes & forms > processes > Task assignment

A user task's `assignee` is a role / candidate-group name, or the literal **`assignee: personal`** to route the task to the **record owner's** inbox (requires the trigger entity to declare a `personal:` relation — see [scoped surfaces](#scoped-surfaces--roles)), or a **relation walk** off the trigger record:

```yaml
- name: approve
  kind: userTask
  args:
    assignee: { path: employee.manager, fallback: manager }
    form: ApproveRequest
```

Every segment of `path` is a **to-one relation** — the first of the trigger entity, each further one of the previous target — and the walk ends at an entity that declares `identity`, which is what maps a record to a login. A **cross-model** relation may only be the **last** segment: a projection carries the target's own properties but not its relations, so there is nothing to walk on from there. A conforming generator validates every hop when the file is read, so a dangling segment is reported then rather than when the process runs.

`fallback` is **required** and names the candidate group. The walk is resolved when the task is reached, not when the process starts — so a relation an earlier step of the same process set is visible — and when it resolves to nobody (a null hop, a missing record, a blank identity) the task is created **unassigned** and the fallback group can still claim it. That is what makes the unresolvable case total: a resolver path can never mint a task nobody can see.

**Anchor:** Scoped surfaces & roles > Personal and partner surfaces

A user task can also be routed to the record owner's inbox with the literal `assignee: personal`, which resolves the owner through the `personal:` relation, or to whoever a relation walk off the record names — `assignee: { path, fallback }`, whose walk likewise ends at an `identity`-declaring entity (see [processes](#task-assignment)).

**Anchor:** Appendix A: DSL index

| [task assignment](#task-assignment) | route a user task to a role, the record owner, or a relation walk |
