# Deleting a document retires its running process

- **Status:** released in [1.9](../versions/1.9.md)
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7074
- **Implementation:** [eclipse-dirigible/dirigible#7087](https://github.com/eclipse-dirigible/dirigible/pull/7087)
  (the construct), [eclipse-dirigible/dirigible#7169](https://github.com/eclipse-dirigible/dirigible/pull/7169)
  (the refusal enforced on the personal surface as well)

## The problem

A process with an entity trigger runs *for* a record: the instance is stamped on the row, its user
tasks open a form over the row, its service tasks write the row. `abortOn:` retires that instance
when the record is voided or cancelled - it listens to the record **transitioning into** a terminal
status and cancels the whole in-flight instance, so no Inbox task outlives its document.

A record has a second way out that is not a transition. It is deleted. Nothing is published on the
status channel, `abortOn` does not fire, and the instance keeps running for a row that no longer
exists:

```
SalesInvoiceApproval { trigger: { onCreate: SalesInvoice } }

POST   SalesInvoice                 -> 11, DRAFT; task "Sales Invoice Approval - Approve" (ref 11) raised
DELETE SalesInvoice 11              -> accepted
GET    SalesInvoice 11              -> not found
Inbox                               -> the Approve task for ref 11 is still there
```

The task is claimable. It opens a form in which every field reads as empty. It still offers Approve
and Reject, and completing it drives the flow's next step - a status write, a notification, a posting -
over nothing. A clerk cannot tell the task is dead; the application says nothing.

A model cannot express either of the two sane outcomes today: that deleting the document ends its
flow, or that the document cannot be deleted while its flow runs. The outcome it silently gets is
neither.

## The proposed shape

One key on the process, beside `abortOn:`:

```yaml
processes:
  - name: SalesInvoiceApproval
    trigger: { onCreate: SalesInvoice }
    abortOn: { status: [4, 5], then: markVoid }   # cancelled / rejected - a TRANSITION retires the flow
    whenDeleted: refuse                           # abort (the default) | refuse - a DELETE retires it, or is refused
    steps:
      - { name: approve,  kind: userTask,    args: { assignee: approver, form: ApproveInvoice, next: done } }
      - { name: markVoid, kind: serviceTask, args: { setRelationField: Status, value: 8 } }   # abort-only cleanup
      - { name: done,     kind: end }
```

- **`abort`** - the default; the key may be omitted. Deleting the record goes through, and the
  instance this process stamped on the record is cancelled with it: pending user tasks withdrawn,
  parked waits and armed timers cancelled.
- **`refuse`** - deleting the record is rejected while the instance this process stamped on it is
  still running. The record becomes deletable once the flow has ended - completed, or aborted by an
  `abortOn` transition.

`whenDeleted` decides which of the two. It cannot ask for the third: no task may point at a deleted
row either way.

## Expected behaviour

For every process that declares an entity trigger, a conforming generator produces a reaction to the
trigger entity's **deletion** - independent of `abortOn`, which reacts to its transitions, and
independent of the value of `whenDeleted`:

- The reaction finds the instance **this** process stamped on the deleted record. No stamp, or an
  instance that has already ended, is a no-op - the reaction is fail-soft, exactly like `abortOn` and
  `wait`.
- A still-running instance is **cancelled** as a whole: its pending user tasks are withdrawn, its
  parked waits and armed boundary timers cancelled, and the cancellation records that the record the
  flow ran for was deleted. A cancellation that did not take effect is reported, never swallowed - a
  flow that keeps running over a deleted row is the defect, however the row went.

`whenDeleted: refuse` adds a guard in front of the deletion on **every generated interface** through
which the record can be deleted - the ordinary surface, and the personal and partner surfaces where
one exists:

- While the instance this process stamped on the record is still running, the deletion is rejected
  as a **conflict** the caller can distinguish from a generic failure, with a message naming the
  record's entity and the process it is still in, and saying what to do about it. The reference
  implementation's message reads: *"This Sales Invoice is still in its Sales Invoice Approval flow -
  complete or cancel it before deleting"*.
- Nothing is deleted, and the instance is untouched.
- An instance that has ended - completed, or aborted through `abortOn` - does not block the deletion.
  A record that was never stamped by this process is deletable.

When several processes with `refuse` are triggered by the same entity, each guards the deletion
independently: the record is deletable only when none of them has a running instance.

## Edge rules

- Valid only on a `processes[]` entry that declares an entity `trigger:`. On a process started
  otherwise - a schedule, an arriving message, a step of another flow - the key MUST be rejected,
  whichever value it carries: there is no row whose deletion could mean anything to the instance.
- Values are `abort` and `refuse`. Any other value MUST be rejected with a message that names the
  process and lists the two.
- The key omitted is `abort`. The behaviour is not conditional on the key: cancelling the instance
  of a deleted record is what a conforming generator does for every entity-triggered process, because
  a flow over a row that is gone is never right. The key exists to choose `refuse`.
- **`refuse` binds the generated interfaces, not the row.** A deletion that reaches the row by another
  path - a cascade from a master whose composition child this is, a reaction, a scheduled purge - is
  not refused; the cancelling reaction applies, so the invariant holds either way. This is the same
  boundary the format draws for an immutable master: the refusal covers every user surface the
  generator produces and does not extend to system writes.
- **The cleanup `then:` of `abortOn` does not run on the delete path.** `abortOn.then:` names a
  service task that writes the record (`setField` / `setRelationField`) after the transition that
  aborted the flow. On a deletion there is no record left to write, so the instance is cancelled
  outright, with no cleanup step. An author who needs something recorded about the deletion binds a
  reaction to the entity's `onDelete` event, as for any other deletion.
- `abortOn` and `whenDeleted` are independent and compose freely: the one listens to the status
  channel, the other to the deletion, and a process may declare either, both or neither. Neither
  requires the other; `whenDeleted` does not require a `function: EntityStatus` relation.
- On a process whose trigger is itself `onDelete: <Entity>` the key is accepted (the process has an
  entity trigger) but has no observable effect: the instance starts after the row is gone, so at
  the moment of the deletion there is no running instance to cancel or to refuse over.
- The refusal is decided at the moment of the deletion request against the instance's current state.
  A flow that ends between the guard and the deletion has simply made the record deletable.
- **Relation to `whenMasterDeleted` (proposal 0033, spec PR #66).** The two are siblings, one per
  consequence of a delete: `whenMasterDeleted` on a composition relation decides what happens to the
  **rows** a deleted master owns (cascade, or refuse while they exist); `whenDeleted` on a process
  decides what happens to the **flow** the deleted record was in (abort, or refuse while it runs).
  Both default to the outcome that leaves nothing behind, both offer `refuse` as the alternative, and
  neither respecifies the other. A cascade from `whenMasterDeleted: cascade` deletes child rows below
  the generated interfaces, so a child's `whenDeleted: refuse` does not stop it - the child's flow is
  aborted, per the rule above.
- **The current text.** Version 1.6 does not contradict the shipped behaviour; it is silent. Its
  `abortOn` section opens with "A running process should not outlive its document" and closes by
  calling `abortOn` "the structural answer to orphaned inbox tasks", while `abortOn` fires on a
  transition only - a reader taking the sentence at face value believes the orphan hole is closed and
  it is not. The Specification text below narrows that closing sentence to the status path and adds
  the delete path beside it.

## Prior art / workarounds

Without the construct an author writes the reaction by hand: a listener on the trigger entity's
deletion that looks up the stamped instance and cancels it - once per process, remembered far less
often than that, and with no way to express the refusal at all short of a hand-written guard in the
delete path of every generated surface. The reference implementation shipped the cancelling reaction
for every entity-triggered process and the refusal as an opt-in in eclipse-dirigible/dirigible#7087;
the refusal was found unenforced on the personal surface - a requester could delete their own record
and the guard was not there - and closed in #7169, which is why the Specification text names every
generated interface rather than "the API".

Workflow engines offer the cancellation primitive (terminate an instance by id) and nothing above it:
they do not know which record an instance runs for, so the correlation, the stamp and the refusal are
the model's to state.

## Specification text

**Anchor:** Processes & forms > processes > abortOn — cancel the instance on a terminal status. The
closing paragraph of that section is amended as below, and the new `whenDeleted` subsection follows
it, before `trigger`.

*Amended closing paragraph of `abortOn`* (replaces "This is the structural answer to orphaned inbox
tasks: cancel a review the moment its document is voided elsewhere."):

Like `wait`, `abortOn` requires the process to declare a `trigger:` (correlation rides the instance
identifier stamped on the record) and is **fail-soft** — no running instance is a no-op. It is the
structural answer to orphaned inbox tasks on the **status** path: cancel a review the moment its
document is voided elsewhere. A record also leaves a running flow by being **deleted**, which is not
a transition and which `abortOn` does not see — that path is [`whenDeleted`](#whendeleted--retire-the-instance-when-the-document-is-deleted).

#### whenDeleted — retire the instance when the document is deleted

A deleted record must not leave its flow running: a user task over a row that no longer exists opens
an empty form and can still be completed, driving the flow over nothing. `whenDeleted:` on a process
with an entity trigger chooses between the two safe outcomes:

```yaml
processes:
  - name: SalesInvoiceApproval
    trigger: { onCreate: SalesInvoice }
    whenDeleted: refuse      # abort (the default) | refuse
```

- **`abort`** (the default; the key may be omitted) — deleting the record goes through and cancels
  the instance this process stamped on it: pending user tasks withdrawn, parked waits and armed
  boundary timers cancelled.
- **`refuse`** — deleting the record is rejected while the instance this process stamped on it is
  still running. The record becomes deletable once the flow has ended — completed, or aborted through
  [`abortOn`](#aborton--cancel-the-instance-on-a-terminal-status).

Either way no task may point at a deleted row. `whenDeleted` decides which of the two outcomes; it
cannot ask for the third.

> **Normative.** For every process that declares an entity `trigger:`, a conforming generator MUST
> react to the trigger entity's deletion by cancelling the instance this process stamped on the
> deleted record, if that instance is still running — its pending user tasks, parked waits and armed
> timers with it — whatever `whenDeleted` says. The reaction MUST be fail-soft: no stamp, or an
> instance that has already ended, is a no-op. A cancellation that did not take effect MUST be
> reported. The `then:` cleanup of `abortOn` MUST NOT run on this path — there is no record left for
> it to write.

> **Normative.** With `whenDeleted: refuse`, a deletion of the trigger entity's record requested
> through **any** interface the generator produces — including a personal or partner surface — MUST
> be rejected while the instance this process stamped on the record is still running, as a conflict
> the caller can distinguish from a generic failure, with a message naming the record's entity and
> the process it is still in. Nothing MUST be deleted and the instance MUST be untouched. An instance
> that has ended MUST NOT block the deletion. The refusal covers the generated interfaces and MUST NOT
> extend to a deletion that reaches the row otherwise — a cascade from a master, a reaction — where
> the cancelling reaction above applies instead.

`whenDeleted` is valid only on a process with an entity trigger; on a process started by a schedule,
a message or another flow it MUST be rejected with either value. Its values are `abort` and `refuse`;
any other value MUST be rejected. `whenDeleted` and `abortOn` are independent: the one reacts to the
deletion, the other to a transition, and neither requires the other. What a deletion does to the
**rows** a record owns is the composition relation's `whenMasterDeleted`, not this key.

## DSL index

| Construct | What it gives you |
| --- | --- |
| [`whenDeleted`](#whendeleted--retire-the-instance-when-the-document-is-deleted) | cancel the running instance when the document is deleted (the default), or refuse the delete while it runs |
