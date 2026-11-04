# A setter step may write a member of a to-one relation of its trigger entity

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/131, https://github.com/eclipse-dirigible/dirigible/issues/7934
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7936

## The problem

A process's setter steps - `setField`, `clearField`, `setRelationField` - write the **trigger
record** only. 1.11 says so in as many words: "the record they act on is the trigger record". A
very common business flow is started by one record and has to move **the record it belongs to**:
an approval request moving its order to `APPROVED`, a shipment moving its order to `SHIPPED`, a
claim's assessment moving the claim to `ASSESSED`.

That move cannot be declared. The author writes a `delegate` whose whole body is one status write on
another entity, and writes it again for every such flow. The hand-written write also bypasses what
the model declares about the target: its legal status moves, the timestamps it stamps on entering a
status, and the event other flows react to. And because 1.11 makes a status the flow's column only
when the target's own setter writes it, the order's status stays open to a direct edit that skips
the approval flow altogether.

## The proposed shape

```yaml
processes:
  - name: approvalRequest
    trigger: { onCreate: ApprovalRequest }        # ApprovalRequest has a manyToOne `order`
    steps:
      # one hop: the Status of the order this request belongs to, named in the ORDER's nomenclature
      - { name: markOrderApproved, kind: serviceTask, args: { setRelationField: order.Status, value: APPROVED, next: note } }
      # the same hop for a text field of the order
      - { name: note,              kind: serviceTask, args: { setField: order.approvalNote, value: "approved by request", next: done } }
      - { name: done,              kind: end }
```

## Expected behaviour

The setter's target may be `<relation>.<member>`: exactly **one hop** through a to-one relation of
the trigger entity. The step writes `<member>` of the record that relation points at, under the same
rules the setter has on the trigger record - the member's type, the value's form, and for
`setRelationField` the status named by the **target's** seeded name or id.

The write is the **target record's own write**, not a side door around it:

- the target's legal status moves are enforced, and a refused move fails the step, where `retry` /
  `onError` apply as they do to any failed step;
- the target's timestamps tied to entering a status are stamped;
- the checks the target declares over the status it enters judge the write, as they judge the
  target's own setter;
- a status move publishes the target's status-change event after the write commits, so a flow on
  the target (an `onTransition` trigger, a transition-driven reaction) reacts to it as it would to
  the target's own setter.

A trigger record whose relation is unset at run time has no target: the step writes nothing, reports
it (naming the process, the step and the relation), and the flow continues.

A status written one hop away makes that status **the flow's column** on the target, exactly as a
status written by the target's own setter does: a direct create or update of it through a generated
surface is refused as a conflict (409 over HTTP).

## Edge rules

- One hop only: a path with more than one dot is rejected.
- The first segment MUST be a to-one relation (`manyToOne` / `oneToOne`) of the trigger entity,
  declared in the same model. A to-many relation, a field, an unknown name and a cross-model
  relation are rejected.
- The member MUST exist on the target and be writable by that setter under its existing rules; a
  member of a type the setter cannot write is rejected with the plain setter's own reason, and a
  status name the target's nomenclature does not carry is rejected.
- A status move no declared legal move of the target can reach is rejected at authoring time, as
  it is for the target's own setter.
- Every refusal names the process, the step, the path and the reason.
- A setter without a dot is unchanged, and so is everything a model without a dotted setter
  generates.

## Prior art / workarounds

A hand-written `delegate` that loads the related record and writes the field, bypassing the
target's declared rules and leaving its status open to a direct edit. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/7936.

## Specification text

**Anchor:** Processes & forms > processes > Service tasks, appended after the `> **Normative.**`
blockquote that closes the `setField` / `clearField` rules; that blockquote's "the record they act
on is the trigger record" is amended as stated at the end. Processes & forms > processes > A status
the flow writes is the flow's column, one sentence appended to its first paragraph and one to its
normative blockquote.

A setter may write **one hop away**: the target of a `setField`, `clearField` or `setRelationField`
may be `<relation>.<member>`, a member of the record a to-one relation of the trigger entity points
at. A flow started by one record can then move the record it belongs to:

```yaml
- { name: markOrderApproved, kind: serviceTask, args: { setRelationField: order.Status, value: APPROVED, next: done } }
```

> **Normative.** A setter target containing a dot MUST be `<relation>.<member>`, where `<relation>`
> is a to-one relation of the trigger entity declared in the same model and `<member>` is a member
> of its target that the same setter could write on that entity. A path with more than one dot, a
> first segment that is not such a relation (a to-many relation, a field, a cross-model relation or
> an unknown name), a member the target does not declare or the setter cannot write, and a status
> name the target's nomenclature does not carry MUST be rejected at authoring time, the report
> naming the process, the step and the path. A `setRelationField` one hop away over the target's
> status MUST be checked against the target's declared legal status moves as the target's own
> setter is. At run time the step MUST write the member through the target record's own targeted
> write, so that the target's status rules apply and a refused move fails the step, its timestamps
> tied to entering a status are stamped, the checks it declares over that status judge the write,
> and a status move publishes the target's status-change event after the write commits. A trigger
> record whose relation is unset MUST make the step write nothing and report it, naming the
> process, the step and the relation; the flow continues. A setter target without a dot MUST
> behave as before.

In the `setField` / `clearField` blockquote, "the record they act on is the trigger record" becomes
"the record they act on is the trigger record, or the record one hop away its path names".

Appended to the first paragraph of *A status the flow writes is the flow's column*: a
`setRelationField` step that writes the status **one hop away** has declared, by that step, that
the flow owns the status of the target entity in the same way.

Appended to its normative blockquote: a `setRelationField` step over `<relation>.<status>` MUST make
the target entity's status relation flow-owned exactly as a `setRelationField` step of a process
triggered by the target itself does.

## DSL index

No new row. The `clearField` row gains "or a member of a to-one relation's target, one hop away"
after "the trigger record".
