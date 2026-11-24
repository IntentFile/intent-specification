# A transition asks for input and writes it with the status

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7909
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8891

## The problem

A `transitions` entry is a one-click status move. A move the law requires to carry data has no way
to collect it. Annulling an issued invoice must record a **reason**, and the invoice is locked by
`immutableWhen` from the moment it is issued:

```yaml
entities:
  - name: SalesInvoice
    immutableWhen: "Status == ISSUED || Status == SENT || Status == VOIDED"
    fields:
      - { name: voidReason, type: string, length: 500 }
transitions:
  - { name: VoidSalesInvoice, forEntity: SalesInvoice, from: [ISSUED, SENT], setStatus: VOIDED }
```

The clerk cannot type the reason on the record before pressing Void - the lock refuses the edit, as
it should - and the button asks for nothing. A `generates` action has `prompt:` for "ask, then act",
but its answers are written onto the record it creates, not onto the record being moved.

## The proposed shape

`prompt:` on a transition, with the `generates` prompt's grammar, naming properties of the record
being moved:

```yaml
transitions:
  - name: VoidSalesInvoice
    forEntity: SalesInvoice
    from: [ISSUED, SENT]
    setStatus: VOIDED
    prompt:
      - { field: voidReason, required: true }
      - { field: ReplacedBy }            # an optional to-one
```

## Expected behaviour

- **The button asks first.** A prompted transition opens an input form instead of the plain
  confirmation, one control per entry, typed from the record's own field or relation - a to-one as a
  picker over its target.
- **The answers are written with the move.** They are written in the same write as the new status,
  so the record is never seen in the new status without them, nor with them in the old one. The
  user-write lock of `immutableWhen` does not refuse them: they are the transition's write, not an
  edit.
- **The gated checks read them.** A check gated on the target status - a `requiredWhen` on the reason
  at VOIDED - is evaluated on the record carrying the answers, so it can enforce them for every writer
  that moves the record there.
- **A required answer left out refuses the move**, naming the field, for any caller of the
  transition - not only in the form - and the record does not move.
- **An optional answer left out** leaves the field as it is stored.

## Edge rules

- An entry names a field or a to-one relation of the transition's `forEntity`, once.
- It does not name what the platform or the move writes: the status relation, the primary key, a
  `number:` field, a `stampOn: { status }` field. A date stamped with the move is the field's own
  `stampOn: { status: ... }`, not a prompt.
- A `timestamp` field is not prompted, as for `generates`.
- `default:` is not allowed: on a `generates` prompt it names a property of the source, and the record
  a transition would default from is the record it writes.
- A field that only the move should write is declared `readOnly: true` as any system-written field is;
  `prompt:` does not change how the field is edited elsewhere.

## Prior art / workarounds

A `requiredWhen` gated on the target status alone refuses the move but leaves the user no way to
supply the value through the lock. A `call:` action to a hand-written endpoint can collect the value
and write the record, outside the transition's guards and its `-transitioned` event. Making every
prompted field read-only implicitly was considered; the existing `readOnly: true` already says it,
and a field may legitimately be both editable on a draft and asked again by a move.

## Specification text

**Anchor:** Declarative glue > transitions — guarded status flips, a new subsection after
"personal — the owner runs the transition".

#### prompt — the transition asks for input

A transition MAY declare `prompt:` - the input a move must carry, asked when the button is pressed.
Each entry names a field or a to-one relation of the record being moved, with the
[`generates` prompt](#prompted-input--prompt) grammar: `field`, and `required`.

```yaml
transitions:
  - name: VoidSalesInvoice
    forEntity: SalesInvoice
    from: [ISSUED, SENT]
    setStatus: VOIDED
    prompt:
      - { field: voidReason, required: true }
      - { field: ReplacedBy }            # an optional to-one
```

The answers are written in the same write as the status, so an `immutableWhen` lock - a rule about
user edits - does not refuse them, and a [check](#checks--declarative-validations) gated on the target
status reads them. Declare a field only the move writes `readOnly: true`; a date stamped with the move
is a [`stampOn: { status }`](#stampon--status---when-a-record-entered-a-status) field, not a prompt.

> **Normative.**
> A prompt entry MUST name a field or a to-one relation of `forEntity`, at most once, and MUST NOT
> name its `function: EntityStatus` relation, its primary key, a `number:` field, a
> `stampOn: { status }` field or a `timestamp` field; a transition prompt entry MUST NOT declare
> `default:`. A conforming generator MUST present the inputs before the move, and its transition
> endpoint MUST refuse a request that leaves a `required` input empty with a validation error naming
> that field, after the `from` and `when` guards and before anything is written. The answers MUST be
> written in the same write as the target status - the write that publishes the `-transitioned`
> event - and the checks gated on the target status MUST
> be evaluated on the record carrying them. That write MUST NOT be refused by `immutableWhen`. An
> optional input left empty MUST leave the stored value unchanged. A transition without `prompt:`
> MUST generate exactly as before.

## DSL index

| Construct | What it does |
| --- | --- |
| [`transitions[].prompt`](#prompt--the-transition-asks-for-input) | the transition asks for the record's own fields before the move and writes the answers with the status; a required one left out refuses the move |
