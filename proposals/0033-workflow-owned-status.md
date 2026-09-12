# A status the flow writes is the flow's column

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7339

## The problem

An entity whose status is driven by a `processes:` flow gets its `function: EntityStatus` relation
generated as an ordinary writable property on every REST surface. Nothing in the generated
create/update has any notion that the column is the flow's, so a direct write moves the document
anywhere it likes:

```
PUT /.../VacationRequestController/7
{ ..., "Status": 3 }          # 3 = APPROVED
-> 200
```

The record is now APPROVED. The capacity check never ran, the delegate that charges the employee's
leave account never ran, no manager ever saw a task. The document reads approved and the accounts do
not know about it - the exact divergence the approval flow exists to prevent. It is not one model's
problem: a payroll run can be put into POSTED without a payslip, a timesheet into APPROVED without
an approver.

Neither existing construct closes it. `immutableWhen:` locks the way OUT of an outcome, not the way
IN: a DRAFT record is mutable by definition - that is what a draft is - so the DRAFT -> APPROVED jump
passes it and lands. A `transitions[]` button is an ADDITIONAL guarded endpoint; it does not close
the plain `PUT` generated beside it. And a `lifecycle:` graph only holds where the author declared
one, and then only refuses moves no edge names - DRAFT -> APPROVED is normally a legal edge, just
not one a person gets to take by hand.

## The proposed shape

No new key. The declaration already exists: a `processes:` flow that writes the status is the
statement that the flow owns it.

```yaml
entities:
  - name: VacationRequest
    fields:
      - { name: id, type: integer, primaryKey: true, generated: true }
    relations:
      - { name: Status, kind: manyToOne, to: RequestStatus, function: EntityStatus, init: 1 }

processes:
  - name: Approval
    trigger: { onCreate: VacationRequest }
    steps:
      - { name: decide,  kind: userTask, args: { assignee: manager, form: DecideRequest } }
      - { name: approve, kind: serviceTask, args: { setRelationField: Status, value: 3 } }
      - { name: end,     kind: end }
```

## Expected behaviour

A conforming generator refuses a create or update that sets or changes that column, on every
generated surface, naming it:

```
409  "'Status' changes through the workflow, not a direct edit"
```

The flow's own writers are unaffected. A `setRelationField` step and a `transitions[]` endpoint
write the column directly through the model's own targeted-write path, never through a create or an
update of the whole record, so nothing the model declares loses a way to move the status.

Reading the flip side: the status is derived state owned by the flow, the same class of column as an
aggregate or a roll-up target, which a conforming generator already refuses a user write to.

## Edge rules

- **An absent value is not a change.** A caller sends the fields its form edits; the stored status is
  kept rather than refused - and rather than erased, which a whole-record write would otherwise do.
- **A create may carry the declared start.** With `init:` declared, a create naming exactly that
  status is accepted (it is where the record starts); any other value is refused, and so is any value
  at all when no start is declared. A record cannot be created in the middle of its own flow.
- **A `transitions:` button does not claim the column.** It is a user action over the status - the
  declared way a person moves it by hand - and the construct that guards every OTHER hand write is
  `lifecycle:`. Closing the plain write here would make an unmodeled move reachable from nowhere and
  the state machine's refusal observable from nowhere: a different construct removed rather than this
  one delivered.
- An entity whose status no flow writes generates exactly as before.

## Prior art / workarounds

Today the only way to approximate it is `immutableWhen:` over every status the flow can reach, which
freezes the whole record in those statuses (the way out, not the way in) and still leaves the jump
from DRAFT open. The alternative in the field is to not expose the generated controllers at all,
which gives up the REST surface the model exists to produce.
