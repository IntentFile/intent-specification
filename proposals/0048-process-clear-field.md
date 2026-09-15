# A process can clear a field it wrote

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7386
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7388

## The problem

A `setField` service task writes a literal into a `string` / `text` field of the record the process
runs on. Its `value` may not be blank - a blank value is refused at authoring time, because
`value: ""` reads as "I forgot to fill this in" rather than as an intention. The consequence is that
a process can write any string into a field **except an empty one**, and therefore has no
declarative way to take back a field it wrote earlier.

The case this comes from is the error route. A flow that calls out records the failure text with
`setField: errorMessage, value: "{error}"` (the whole-value placeholder proposed in
[0033 - a step's failure is part of the model](https://github.com/IntentFile/intent-specification/pull/71))
and moves the record to a failure status. An operator fixes the cause and re-drives the instance; it
succeeds; the record ends in a **success** status - still carrying the previous failure's
explanation, because nothing in the model can erase the column:

```
Tenant t-0042   Status: Provisioned   ErrorMessage: "schema t_0042 already exists"
```

The natural authoring - a first step that clears the field - is a parse error:

```yaml
- { name: resetError, kind: serviceTask, args: { setField: errorMessage, value: "", next: provision } }
```

The only way out was a `delegate:` step whose whole body was one "set this property to nothing" -
hand-written code for something the model otherwise expresses completely, and written once per
process that shares the pattern.

## The proposed shape

A service task's `args` carries `clearField: <field>`, naming the field and nothing else:

```yaml
processes:
  - name: TenantProvisioning
    trigger: { onCreate: Tenant }
    steps:
      - { name: resetError,    kind: serviceTask, args: { clearField: errorMessage, next: provision } }
      - { name: provision,     kind: serviceTask, args: { delegate: tenants.Provision, retry: { count: 3, every: PT30S }, onError: recordFailure, next: markReady } }
      - { name: markReady,     kind: serviceTask, args: { setRelationField: Status, value: Provisioned, next: done } }
      - { name: done,          kind: end }
      # the error route (proposal 0033): {error} is the final attempt's message
      - { name: recordFailure, kind: serviceTask, args: { setField: errorMessage, value: "{error}", next: markFailed } }
      - { name: markFailed,    kind: serviceTask, args: { setRelationField: Status, value: Failed } }
```

The pair reads as it runs: the route *writes* the failure with `setField`, and the head of the flow
*clears* it with `clearField`, so an instance re-driven to success leaves the record clean.

The blank-value refusal on `setField` **stays**. `clearField` is an explicit key precisely so that
an erasure cannot be confused with an unfilled value.

## Expected behaviour

A conforming generator produces, for a `clearField` step, the same kind of write it produces for a
`setField` step, assigning **nothing** to the named field:

- it acts on the record the process runs on - the process's `trigger` entity, correlated exactly
  as a `setField` is;
- it is a **targeted single-column write**: every other column of the record, and any concurrent
  write to it, is left untouched;
- it is observable wherever a `setField` step's write is observable, and silent wherever that write
  is silent. In particular it raises no ordinary change event (a targeted writer does not, by
  design - see the glue preamble), so it cannot re-trigger an `onUpdate` glue entry, and an
  `onStepCompleted` observer of the step sees the field already empty;
- the field reads as **empty / absent** afterwards - the same value a record has in a field nobody
  ever filled.

## Edge rules

- **Only a `serviceTask`.** `clearField` on any other step kind is rejected at authoring time, the
  report naming the process and the step ("uses clearField but is not a serviceTask").
- **Only a field of the trigger entity.** The process MUST declare a `trigger:` - without one there
  is no record to clear a field on, and the step is rejected saying so. The named field MUST be a
  field the trigger entity declares; anything else is rejected as "not a field of" that entity.
  A relation is not a field, so naming one is rejected the same way - the format has no erasure
  twin of `setRelationField`; a status relation is moved with `setRelationField`, never emptied.
  A field of another entity cannot be named at all, because the key takes a field name, not a
  path.
- **Only a `string` / `text` field.** A number, a date, a boolean is rejected ("must be a string/text
  field"): an erasure is the counterpart of a literal write, and `setField` writes only literal
  strings; what "empty" would mean for a number or a date is not something the model says.
- **No `value`.** A `value:` next to `clearField` is rejected ("takes no value - it erases the
  field; write one with setField"). Conversely a `setField` with a blank `value` remains rejected
  ("must declare a value") - `clearField` is the erasure, not a relaxation.
- **One field, one way.** `clearField` MUST NOT be combined on one step with `setField` or
  `setRelationField` ("cannot be combined ... a step writes one field, one way"), nor with
  `delegate` or `notify` - the existing rules that keep those steps' work single list it, so the
  report a `delegate` or `notify` step gives for an extra action names `clearField` alongside
  `setField` and `setRelationField`. A sending step stands alone (the existing normative
  rule on `notify`), and a `clearField` next to it is an extra action like any other.
- **Accepted wherever a `setField` is.** That includes an `abortOn` `then:` cleanup step - "clear
  the working note on the abort path" is exactly that shape - which the current version text lists
  as `setField` / `setRelationField` only.
- **A recognised key.** `clearField` joins the step vocabulary, so a near-miss (`clearFeild`) is
  reported as an unknown key that names the real one, under the existing rule for unrecognised
  keys.
- **`required` is not examined.** The reference implementation states no rule about clearing a
  field that is `required`, and this proposal adds none: the erasure is a write like any other and
  is answered by the same store that answers every write. Authors should not clear a field the
  entity requires; a rule refusing it at authoring time can be proposed on its own evidence.
- **Not a step-data key.** Clearing declared step data (`vars:` / `clearAfter`) is a different
  construct, shipped alongside proposal 0033 and out of scope there and here; `clearField` erases a
  **column of the record**, not a process variable.

Where the current version text is silent rather than contradicted: the `Service tasks` paragraph
lists `setField` / `setRelationField` / `notify` / `delegate` and no erasure; the normative rule on a
`notify` step, the `abortOn` `then:` bullet and the notify chapter's "stands alone" sentence list the
actions a step may carry and omit `clearField`. The Specification text below replaces those
sentences so that each names it. The version text also does not state `setField`'s own rules -
trigger entity only, `string` / `text` only, non-blank `value` - which the reference implementation
enforces; the replacement states them once for both keys.

## Prior art / workarounds

Before this key, the erasure was a `delegate:` step whose entire body set the property to nothing -
one hand-written handler per process that needed it, outside the model, invisible to a reader of the
`.intent` file, and a second place to keep in step when the field was renamed. The other workaround
was to leave the stale text in place and let the success status "win", which is what the case above
shows: a success record explaining a failure that did not happen.

The reference implementation ships `clearField` as the same machinery as `setField` end to end -
one setter descriptor with an erasure flag, so every previously generated setter is unchanged - and
verifies it by running the generated step: the flow writes the failure text first, the test reads
it back, and only then does the erasure step run, because a column that reads empty at the end
proves nothing on its own.

## Specification text

**Anchor:** Processes & forms > processes > Service tasks - the paragraph "Service-task shapes: ..."
and the `> **Normative.**` blockquote that follows it are replaced by the text below. The `abortOn`
section's `then:` bullet and the notify chapter's sentence "A sending `serviceTask` stands alone ..."
are amended as stated at the end.

Service-task shapes: `setField` / `clearField` / `setRelationField` (generated handlers that write a
field, erase a field, or flip a status relation on a branch), `notify` (the step's work IS an outbound
message - see [the notify block](#the-notify-block--and-attach-print-sending-the-document-itself)),
and `delegate` (a handler referenced by name with injected `fields` - hand-written, or a generated one
such as a [snapshot generator](#attachments-and-snapshots)). Set a status on the *branch* that reaches
it, never on the shared task, so a reject path does not transit through the approved status.

`setField: <field>, value: <literal>` writes a literal into a `string` / `text` field of the process's
trigger entity; the `value` is required and may not be blank. `clearField: <field>` is its erasure
twin: it names a `string` / `text` field of the trigger entity and takes no `value`, and the step
writes nothing into it, so the field reads as empty afterwards. It exists so a flow can take back a
field it wrote earlier - the failure text an error route records with a `setField` is cleared at the
head of the flow, and an instance re-driven to success does not end in a success status still
carrying the previous failure's explanation:

```yaml
steps:
  - { name: resetError, kind: serviceTask, args: { clearField: errorMessage, next: provision } }
```

> **Normative.**
> `setField` and `clearField` are accepted on a `serviceTask` only, and only in a process that
> declares a `trigger:` - the record they act on is the trigger record. Each MUST name a `string` /
> `text` field the trigger entity declares; a field of another type, a relation, or a name the entity
> does not declare MUST be rejected at authoring time, the report naming the process, the step and
> the field. A `setField` MUST declare a non-blank `value`; a `clearField` MUST NOT carry a `value`
> - the two keys are the write and the erasure, and a blank literal is neither. A step writes one
> field, one way: `clearField` MUST NOT be combined with `setField` or `setRelationField` on the same
> step, nor with `delegate`. A `clearField` is accepted wherever a `setField` is,
> including an `abortOn` `then:` cleanup.
> A `clearField` step's write is a targeted single-column write of the trigger record, observable
> exactly where a `setField` step's write is observable and silent exactly where it is silent; an
> `onStepCompleted` observer of the step MUST see the field already empty.
> A `notify` service task stands alone: it MUST NOT carry another action (`setField`, `clearField`,
> `setRelationField`, `call`, `delegate`) on the same step. Sending is the step's whole purpose, and
> a step that both writes and sends hides which of the two failed.

In `abortOn - cancel the instance on a terminal status`, the `then:` bullet reads: `then:` (optional)
- a single cleanup `serviceTask` (`setField` / `clearField` / `setRelationField`) that runs **only**
on the abort path; it must not be reachable from the main flow. Omitted (or `end`) means terminate
with no cleanup.

In `The notify block`, the sentence reads: A sending `serviceTask` stands alone: `notify` MUST NOT be
combined with another action (`setField`, `clearField`, `setRelationField`, `call`, `delegate`) on
the same step - model the send as its own step and route to it.

## DSL index

| Construct | What it does |
| --- | --- |
| [`clearField`](#service-tasks) | a service task that erases a `string` / `text` field of the trigger record - the `setField` step's erasure twin; takes no `value`, one field per step |
