# A step's failure is part of the model — `retry:`, `onError:` and `{error}`, on a delegate and on a send

- **Status:** draft
- **Issue:** <!-- none yet -->
- **Implementation:** [eclipse-dirigible/dirigible#7056](https://github.com/eclipse-dirigible/dirigible/issues/7056)
  (the send half; the delegate half shipped earlier as
  [#6762](https://github.com/eclipse-dirigible/dirigible/issues/6762))
- **Companion:** [`0012-glue-event-axis.md`](0012-glue-event-axis.md) — the step-event axis a process
  already publishes on; this proposal is about the step's *failure*, not its moments.

## The problem

A process step that calls out — provision a schema, register a client with an identity provider, ask
a partner API, send a document to a customer — fails sometimes, and the format has nothing to say
about it. Everything else about the flow is modelled: which step follows which, how a decision
branches, what a boundary timer does when a task is not worked in time. The one thing that is left
entirely to the platform's defaults is *what happens when the work of a step does not succeed*.

The result is a process that is fully modelled right up to the first failure, at which point the
model stops describing what the application does.

**The send is the sharpest case, because the format already prescribes the failure.** The current
version says of a delivery failure that "a sending process step SHOULD [fail the activity], since the
message is that step's whole purpose" — and that is right. But a failed activity is then handled
however the underlying engine handles a failed activity: some bounded number of automatic attempts,
then an incident recorded against the *job*. The record the process is about is untouched. So a
process ordered the way it reads most naturally —

```yaml
steps:
  - { name: issue,  kind: serviceTask, args: { setRelationField: Status, value: Issued, next: notifyCustomer } }
  - { name: notifyCustomer, kind: serviceTask, args: { notify: { to: customer.email, subject: "...", body: "..." }, next: markSent } }
  - { name: markSent, kind: serviceTask, args: { setRelationField: Status, value: Sent, next: end } }
```

— has an outcome nobody would choose: a mail server that is briefly unreachable leaves a document
that was issued correctly sitting in `Issued` for ever, with **no error message and no failure
status**, because the failure was recorded against the job and nothing on the record mentions it. The
document is not visibly wrong. It is invisibly stalled, which is worse.

The workaround is to make the send the **last** step, so the worst case is an incident about the
message alone. It works, and it constrains process design for a reason that has nothing to do with
the domain: the end of a process becomes the only safe place to put a declared send.

## The proposed shape

Two optional arguments on a service task whose work is such a call, plus one placeholder for reading
the failure back:

```yaml
processes:
  - name: TenantProvisioning
    trigger: { onCreate: TenantApplication }
    steps:
      - name: createSchema
        kind: serviceTask
        args:
          delegate: SchemaProvisioner
          retry: { count: 3, every: PT30S }     # three FURTHER attempts, 30s apart
          onError: recordFailure                # where an exhausted failure routes
          next: notifyOwner

      # the same two keys on a send: its whole work is the message, and a mail server
      # blinks exactly as any of the calls above does
      - name: notifyOwner
        kind: serviceTask
        args:
          notify: { to: owner.email, subject: "Tenant {title} is ready", body: "..." }
          retry: { count: 3, every: PT30S }
          onError: recordFailure
          next: markProvisioned

      - { name: markProvisioned, kind: serviceTask, args: { setRelationField: Status, value: Provisioned, next: end } }

      # the error route: {error} is the FINAL attempt's message
      - { name: recordFailure, kind: serviceTask, args: { setField: failureMessage, value: "{error}", next: markFailed } }
      - { name: markFailed,    kind: serviceTask, args: { setRelationField: Status, value: Failed, next: end } }
      - { name: end, kind: end }
```

## Expected behaviour

- **`retry: { count: <n>, every: <ISO-8601 duration> }`** — the step is re-attempted `count`
  **further** times after the first, spaced by `every`. `count` is an integer >= 1; `every` uses the
  same duration vocabulary as a boundary timer's `after`. The attempts are the platform's own
  re-execution of the step, so each failed attempt's partial writes are undone before the next one
  runs.
- **`onError: <step | end>`** — where the **exhausted** failure routes; with no `retry`, the first
  failure is already the exhausted one. Routed and validated exactly like a decision branch, and the
  main flow is routed around the error steps with `next`, exactly as with decision branches.
- **`{error}`** — the failure message of the attempt that routed. A `setField` value of exactly
  `{error}` — the whole value, nothing around it — writes it onto the record the process is about.
- **A step that declares neither keeps today's behaviour**: the failure is the platform's own, and an
  intent that uses none of this is unaffected.
- The writes on the error route commit; the intermediate re-attempted failures do not. This is the
  point of routing rather than retrying for ever: the record ends up carrying *why*.
- **The message a conforming generator makes readable as `{error}` MUST name the cause**, not only
  the step. It is the only account of the failure the record will carry, and "the message could not be
  sent" tells an operator nothing they did not already know from the status.

## Edge rules

Both keys apply to a **`delegate:`** and to a **`notify:`** service task — the two shapes whose work
is a call that can fail transiently and whose failure nobody is waiting on synchronously. Everywhere
else they MUST be an authoring error, reported at generation, because the declaration could never
take effect and a key that is accepted and inert is worse than one that is refused:

- **On a `setField` / `setRelationField` step.** A status write is refused by the model's own
  gates ([`checks`](#), [`lifecycle`](#)), and a gated one is refused *to the person who acted*, in
  the same interaction. Routing that failure away would take the refusal out of their hands, and
  re-attempting a deterministic refusal recovers nothing.
- **On a `call:` step, or a service task carrying neither shape.** Not covered; a hand-written
  handler that wants resilience is bound with `delegate:`.
- **On a fan-out send** — a `notify` carrying `forEach`. A fan-out is fail-soft **per row** by
  construction (the current version: it "MUST NOT fail its activity, because a retry would resend"),
  so the step never fails and neither key could ever fire. The outcome of those deliveries is observed
  per row instead — with the notify block's `outcome:` field and the `onNotifyFailed` event axis.
- **On a step kind other than `serviceTask`.** A user task has boundary timers; a decision, a wait
  and an end have no work to fail.

And:

- `onError` MUST name a declared step or the literal `end`.
- `{error}` MUST be rejected on any step **not reachable from some `onError` route** — nothing else
  ever populates it — and rejected as part of a larger value, since a message concatenated into a
  sentence cannot be read back.
- `retry` MUST be rejected when `count` is not a whole number >= 1, or when `every` is not an
  ISO-8601 duration.

## Prior art / workarounds

Three, all visible in real applications:

- **Make the send the last step.** Contains the damage and distorts the process, as above.
- **Hand-write the resilience into the handler** — a loop with sleeps inside a delegate. It hides
  from the model that the step is re-attempted at all, it cannot route anywhere afterwards, and it
  occupies a worker for the duration.
- **Leave it.** The failure is an incident on a job, in an operations surface, correlated to the
  record by hand.

The second and third are the two halves of the same loss: the application's behaviour on failure
stops being something the model states.

## Specification text

**Anchor:** Processes > Service tasks (after the normative note on a standalone `notify` step)

#### retry / onError — a step's failure is part of the model

A service task whose work is a **call** — a `delegate:` handler, or a `notify:` send — may declare
what happens when that call does not succeed:

```yaml
- name: createSchema
  kind: serviceTask
  args:
    delegate: SchemaProvisioner
    retry: { count: 3, every: PT30S }
    onError: recordFailure
    next: provisioned
- { name: recordFailure, kind: serviceTask, args: { setField: failureMessage, value: "{error}", next: failed } }
```

- **`retry: { count: <n>, every: <ISO-8601 duration> }`** — re-attempt the failed step `count`
  **further** times (an integer >= 1), spaced by `every` (the same vocabulary as a boundary timer's
  `after`). Each failed attempt is undone before the next runs.
- **`onError: <step | end>`** — where the exhausted failure routes, validated and routed like a
  decision branch. With no `retry`, the first failure is the exhausted one. Route the main flow
  around the error steps with `next`, as with decision branches.
- **`{error}`** — the failure message of the attempt that routed. A `setField` value of exactly
  `{error}` writes it onto the record the process is about.

> **Normative.** `retry` and `onError` apply to a `delegate:` and to a non-fan-out `notify:` service
> task. On any other step they MUST be an authoring error rather than an accepted key with no effect:
> on a `setField` / `setRelationField` step, because a status write is refused by the model's own gates
> and a gated one is refused to the person who acted, so routing that failure away would take the
> refusal out of their hands; on a `call:` or bare service task, because neither is covered; on a
> fan-out `notify` (one carrying `forEach`), because a fan-out MUST NOT fail its activity, so neither
> key could ever fire — a fan-out's deliveries are observed with the notify block's `outcome:` field
> and the `onNotifyFailed` axis instead; and on any step kind other than `serviceTask`.

> **Normative.** A step that declares neither key keeps the platform's own failure handling, and an
> intent using neither is unaffected. The writes on an `onError` route MUST commit — the route exists
> so the record carries why the step failed — while the intermediate re-attempted failures MUST NOT.
> The message made readable as `{error}` MUST name the failure's cause and not only the step that
> failed; it is the only account of the failure the record will carry. `{error}` MUST be rejected on a
> step no `onError` route reaches, and as part of a larger value.

## DSL index

| Construct | What it gives you |
| --- | --- |
| [`retry` / `onError`](#retry--onerror--a-steps-failure-is-part-of-the-model) | a declared retry cycle and an error route for a calling step, with `{error}` recording the final attempt's message on the record |
