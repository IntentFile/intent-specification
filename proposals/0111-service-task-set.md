# A setter step may write several members of one record in one write - set:

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/137, https://github.com/eclipse-dirigible/dirigible/issues/8042
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8055

## The problem

1.11: "A step writes one field, one way." A change that is one fact of one record therefore takes
several steps, and every step's write is its own unit of work. Examples:

- the end of a run that releases a held slot and moves to `DONE`;
- a failure that records its text and moves to `FAILED`;
- a re-run that resets the previous error and moves to `RUNNING`.

Between two such steps the record is in a state the model never meant: a slot free while its run
still reads `RUNNING`, or a `FAILED` run without its text. An instance that stops between the two
keeps that state until it is repaired. Each write is also observed separately, so a reader sees
several partial changes - and a flow reacting to the status sees a status-change event per step -
instead of one.

## The proposed shape

```yaml
steps:
  - { name: begin,  kind: serviceTask, args: { set: { errorMessage: null, Status: RUNNING }, next: work } }
  - { name: work,   kind: serviceTask, args: { delegate: RunJob, onError: fail, next: finish } }
  - { name: finish, kind: serviceTask, args: { set: { slotKey: null, Status: DONE }, next: done } }
  - { name: fail,   kind: serviceTask, args: { set: { errorMessage: "{error}", Status: FAILED }, next: done } }
  - { name: done,   kind: end }
```

One hop away, where the format admits a one-hop setter:

```yaml
  - { name: release, kind: serviceTask, args: { set: { job.slotKey: null, job.Status: DONE }, next: done } }
```

## Expected behaviour

A service task may declare **`set:`**, a map of members of **one record** to their values. The step
writes them all as **one** write: one unit of work, observed once. The record's declared constraints
judge the change as a whole: its legal status moves, and the stamps of the status it enters. A status
move among the members publishes one status-change event, after the write commits.

A write the record's lifecycle refuses refuses the whole step: no member is written. In the
example, a restart from `DONE` that the lifecycle does not allow fails the step and leaves the slot
as it was.

## Edge rules

- Each key names what a single-member setter would write: a `string` / `text` field, or a to-one
  relation (a status). Where the format admits a one-hop setter
  (https://github.com/IntentFile/intent-specification/issues/131), a key may name a member one hop
  away.
- All keys MUST address the same record - the trigger record, or the same one-hop target; a mix
  MUST be rejected, the report naming both paths.
- Each value follows its key's single-member setter, with that setter's refusals:

  | Key | Value | Behaves as |
  | --- | --- | --- |
  | a `string` / `text` field | a non-blank literal | `setField` |
  | a `string` / `text` field | `null` | `clearField` |
  | a to-one relation | a seeded name or id | `setRelationField` |

  The value tokens the format admits on `setField` (`{error}` on an error route) stay valid. A value
  its key's setter would refuse MUST be rejected, naming the step and the key.
- `set:` MUST NOT be combined with `setField`, `setRelationField`, `clearField`, `value`, `delegate`,
  `call` or `notify` on the same step, and an empty `set:` MUST be rejected.
- `set:` is accepted on a `serviceTask`, and on a `userTask` for relation keys only - the rule
  `setRelationField` already follows there.
- The single-member setters are unchanged, and a one-key `set:` behaves exactly as the
  single-member setter it names.

## Prior art / workarounds

One step per member, accepting the intermediate states, or a hand-written delegate that writes them
together. Reference implementation: https://github.com/eclipse-dirigible/dirigible/pull/8055.

## Specification text

**Anchor:** Processes & forms > processes > Service tasks. In the shapes paragraph, the setter
entry becomes "`setField` / `clearField` / `setRelationField` / `set` (generated handlers that write
a field, erase a field, flip a status relation on a branch, or write several members of one record
at once)". In the `> **Normative.**` blockquote of the `setField` / `clearField` rules, "A step
writes one field, one way" becomes "A single-member setter writes one field, one way". The text
below follows that blockquote.

`set:` writes several members of one record in **one** write, for a change that is one fact - a run
that releases its slot as it moves to `DONE` never reads as free while still `RUNNING`:

```yaml
- { name: finish, kind: serviceTask, args: { set: { slotKey: null, Status: DONE }, next: done } }
```

> **Normative.** A `serviceTask` MAY instead declare `set:`, a map of members of one record to
> values. Each key MUST name a member a single-member setter could write, and each value MUST be one
> that setter accepts, a `null` standing for the `clearField` erasure; a value the key's setter would
> refuse MUST be rejected, naming the step and the key. Every key MUST address the same record; a mix
> MUST be rejected, naming the paths. The step's writes MUST be one unit of work, judged once by the
> record's legal status moves and the stamps of the status it enters, and observed once - a status
> move among them publishes one status-change event; a refused write leaves every member unchanged.
> `set:` MUST NOT be combined with `setField`, `setRelationField`, `clearField`, `value`, `delegate`,
> `call` or `notify`, and MUST NOT be empty. On a `userTask` it MAY declare relation keys only, as
> `setRelationField` there. A `set:` with one key MUST behave as the single-member setter it names.

## DSL index

| Construct | What it does |
| --- | --- |
| [`set`](#service-tasks) | a service task that writes several members of one record - fields, erasures, a status - in one write, judged once by the lifecycle and observed once |
