# Recording when: `{now}` in a `setField` step, and `stampOn: { status }` on a timestamp or date field

- **Status:** released in [1.13](../versions/1.13.md)
- **Issue:** https://github.com/IntentFile/intent-specification/issues/149, https://github.com/eclipse-dirigible/dirigible/issues/7873, https://github.com/eclipse-dirigible/dirigible/issues/7874
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7917 (`{now}`), https://github.com/eclipse-dirigible/dirigible/pull/7918 (`stampOn: { status }`)

## The problem

"When did this record become APPROVED?" cannot be declared. 1.11's `setField` "writes a literal
into a `string` / `text` field of the process's trigger entity", so a flow that wants an
`approvedAt` beside the status it writes needs a hand-written delegate whose whole body is one
timestamp write. And even with such a step, the stamp has to be repeated **per writer**: a
`transitions:` button, a second process or an action that moves the same status must each repeat
it, or the field stays empty for the records that took that path. 1.11 knows `stampOn` only on a
`number:` field (`create | issue`), and `{now}` only in a send payload.

## The proposed shape

Two shapes; the second states the rule once for every writer of a status:

```yaml
processes:
  - name: approval
    steps:
      # the step records the moment it runs
      - { name: stampApproved, kind: serviceTask, args: { setField: approvedAt, value: "{now}", next: markApproved } }

entities:
  - name: Request
    fields:
      # the record carries the moment it entered APPROVED, whoever moved it there
      - { name: approvedAt, type: timestamp, stampOn: { status: APPROVED } }
```

## Expected behaviour

**`{now}` in a setter.** `setField` accepts the whole-value token `{now}` on a `timestamp` or `date`
field: the step writes the current instant, or the current date. The write is the targeted
single-column write a `clearField` makes, so it publishes no update event of its own.

**`stampOn: { status: X }`.** A `timestamp` or `date` field is stamped with the current moment
whenever its record **enters** status `X`, named by its seeded name or id, on every path that moves
the status: a direct update, a transition button, a process setter, and a create whose `init:` lands
in `X`.

- Entry only: no re-stamp while the record stays in `X`; a later re-entry overwrites the stamp;
  leaving `X` keeps it.
- The stamp is taken after the record's checks and guards: a refused move stamps nothing, and a
  guard that files the record into a status stamps by the status the record lands in.
- The stamped field is read-only: a create drops a value sent for it, a full update keeps the stored
  value whatever it sends, and a form renders it read-only.

## Edge rules

`{now}`:

- a `timestamp` / `date` field takes only `{now}` as its `setField` value;
- `{now}` on a `string` / `text` field is refused - it would store the text `{now}`;
- `{now}` inside a longer value is refused: it is the whole value or nothing;
- every other field type keeps its 1.11 refusal.

`stampOn: { status }`:

- refused on a field that is not `timestamp` / `date`, on a `required: true` field, and beside
  `defaultValue`, `number:`, a calculation, `aggregate:` or `dependsOn`;
- refused on an entity without a `function: EntityStatus` relation, for a status the entity does not
  seed, for an unknown key inside `stampOn`, and when two fields stamp the same status;
- on a status relation owned by another model, only a numeric id is accepted, and it is not checked
  against the owner's seeds.

A file using neither generates byte-identically.

## Prior art / workarounds

A hand-written `delegate:` writing the current instant, placed before every status write and
repeated for each writer. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/7917 (`{now}`), https://github.com/eclipse-dirigible/dirigible/pull/7918 (`stampOn: { status }`).

## Specification text

**Anchor (1):** Processes & forms > processes > Service tasks, appended after the `clearField`
paragraph and its normative block.

`setField` also records when a step ran: on a `timestamp` or `date` field its value is the token
`{now}`, the current instant or the current date, and no literal.

```yaml
- { name: stampApproved, kind: serviceTask, args: { setField: approvedAt, value: "{now}", next: markApproved } }
```

> **Normative.**
> A `setField` naming a `timestamp` or `date` field of the trigger entity MUST declare the value
> `{now}` and no other; the step MUST write the current instant into a `timestamp` field and the
> current date into a `date` field. The write is a targeted single-column write of the trigger
> record, observable exactly where a `clearField` step's write is observable and silent exactly where
> it is silent. `{now}` MUST be rejected on a `string` or `text` field, where it would be stored as
> text, and as part of a larger value. This amends the rule that a `setField` names a `string` /
> `text` field: a `timestamp` or `date` field is accepted with `{now}`; every other type is still
> rejected.

**Anchor (2):** Entities & fields > Document numbering, followed by a new subsection; the fields
attribute table gains a row after `number`.

| `stampOn: { status }` | stamp a `timestamp` / `date` field with the moment its record enters a status (see [stampOn: { status }](#stampon--status----when-a-record-entered-a-status)) |

#### stampOn: { status } - when a record entered a status

A `number:` field is stamped at `create` or at `issue`. A `timestamp` or `date` field may be stamped
when its record enters a status - the field records when, and every writer of the status records it,
not only the one that remembered to:

```yaml
- name: Request
  fields:
    - { name: approvedAt, type: timestamp, stampOn: { status: APPROVED } }
```

The status is named by its seeded name or id, as everywhere a status is referenced. The stamp is
taken on entry: a write that leaves the record in the status does not re-stamp it, a later re-entry
overwrites the stamp, and leaving the status keeps it. A stamped field is read-only; it is the
record's history, not an input.

> **Normative.**
> A conforming generator MUST write the current moment - the current instant into a `timestamp`
> field, the current date into a `date` field - into a field declaring `stampOn: { status: X }`
> whenever its record enters `X` from another status or is created in `X`, on every path that writes
> the record's status: a user update, a `transitions:` button, a process setter, a create whose
> `init:` lands in `X`. It MUST NOT re-stamp a write that does not move the record into `X`, MUST
> overwrite the stamp on a re-entry, and MUST keep it when the record leaves `X`. The stamp MUST be
> taken after the record's checks and guards: a refused move stamps nothing, and a guard that files
> the record into a status stamps by the status the record is stored with. The field MUST be
> read-only: a create drops a value sent for it, an update keeps the stored value, and a form renders
> it read-only. A generator MUST reject as an authoring error: `stampOn: { status }` on a field that
> is not `timestamp` or `date`, on a `required` field, or together with `defaultValue`, `number`, a
> calculation, `aggregate` or `dependsOn`; on an entity without a `function: EntityStatus` relation;
> a status the entity does not seed; an unknown key inside `stampOn`; and two fields stamping the
> same status. On a status relation owned by another model only a numeric id is accepted. A file
> declaring neither `{now}` nor `stampOn: { status }` MUST generate byte-identical output.

## DSL index

| Construct | What it does |
| --- | --- |
| [`stampOn: { status }`](#stampon--status----when-a-record-entered-a-status) | a `timestamp` / `date` field stamped with the moment its record enters a status, on every path that moves the status; read-only |
| [`setField` ... `{now}`](#service-tasks) | a service task that records the current instant or date into a `timestamp` / `date` field of the trigger record |
