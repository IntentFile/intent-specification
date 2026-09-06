# A create-from on the process-step axis, and an opt-in append cardinality

- **Status:** released in [1.6](../versions/1.6.md)
- **Issue:**
- **Implementation:** [eclipse-dirigible/dirigible#6800](https://github.com/eclipse-dirigible/dirigible/issues/6800)

## The problem

"On event E, append a derived row" cannot be expressed. It is an ordinary requirement — a log entry
per step of a workflow, a protocol line per status change, an activity record per delivery — and
every event-driven construct in the format answers a different question:

- `postings` and `posts` write derived rows, but each is idempotent through a back-reference: one
  result per source, permanently.
- `rollups` and `aggregates` recompute a value in an existing row; they never insert.
- an event-driven `generates` create-from is **at-most-once** by construction: the target's
  back-reference to the source is checked before anything is created, so the second event is a no-op.

That leaves a hand-written listener, or an `outbound` departure looped back through `inbound` — a
message round-trip whose only purpose is to defeat the guard, with fail-soft delivery and no
provenance.

Separately, the event-driven create-from binds only to the **source's own lifecycle**
(`onTransition` / `onCreate`), while `notifications`, `integrations` and `outbound` departures bind
to either the lifecycle **or a process step**. So a follow-up document that belongs to a *moment in a
flow* rather than to a status has no trigger to hang off — and neither has one whose source is
written by something that publishes no status transition at all.

## The proposed shape

Two additions to the `generates` `event:` map.

**1. The step axis** — the same `onStepReached` / `onStepCompleted: { process, step }` binding the
other consumers of the event axis already declare.

**2. `mode:`** — the cardinality, `once` (the default, today's behaviour) or `append`.

```yaml
processes:
  - name: ClaimApproval
    trigger: { onCreate: Claim }
    steps:
      - { name: review,   kind: userTask,    args: { assignee: approver, form: ReviewClaim } }
      - { name: activate, kind: serviceTask, args: { setRelationField: Status, value: ACTIVE } }

generates:
  # one LogEntry appended every time the activate step completes — several per Claim, by design
  - name: log-activation
    from: Claim
    to: LogEntry
    event: { onStepCompleted: { process: ClaimApproval, step: activate }, mode: append }
    map:
      Claim: id                # the back-reference: required in both modes
      amount: amount
    defaults:
      step: "activate"         # which moment this row records
      date: now
```

## Expected behaviour

A create-from bound to a step runs when the named process reaches (or completes) the named step, on
the record that process runs on, and creates its target exactly as the lifecycle-bound form does —
same mapping, same items, same completion hook. Nothing about the target changes with the axis.

Under `mode: once` (default) the existing at-most-once behaviour is unchanged: the already existing
target is returned instead of a second one. Under `mode: append` no such lookup happens, so every
delivered event creates another target row, each carrying the back-reference to its source. An
existing file that declares no `mode` and no step binding produces byte-identical output.

## Edge rules

- Exactly one trigger: `onTransition`, `onCreate`, `onStepReached` or `onStepCompleted`. Declaring a
  step binding next to a lifecycle one MUST be rejected.
- A step binding names a process that MUST exist and declare a `trigger`, and a step that MUST exist
  and occupy a moment in the flow (a user task or a service task; a decision, a wait or an end has no
  boundary to observe).
- The process's trigger entity MUST be the entity `from:` declares. A step event is about the record
  its process runs on, and that record is the one the create-from reads.
- A step binding on a source owned by another model MUST be rejected: a process and its steps belong
  to the model that declares them.
- `when:` is optional on the step axis (the step is the moment) and keeps its meaning: a per-record
  filter, evaluated against the source as re-read at delivery.
- `mode:` accepts `once` or `append` only; any other value MUST be rejected rather than read as the
  default. `mode:` without a trigger MUST be rejected — there is no guard on a button to configure.
- The back-reference `map` entry stays REQUIRED in both cardinalities: the dedup key under `once`, the
  created row's provenance under `append`.
- `mode: append` does not change the existing incompatibility of `event:` with `prompt:` — an
  event-driven create-from runs with nobody there to answer a form.
- `append` is the **absence** of a guard, not a state-aware one. It does not express "the target was
  voided, so make another": that needs a state-aware predicate on `mode: once`, since `append` would
  also create a row on every later qualifying event.

## Prior art / workarounds

Today the shape is reachable only outside the format: a hand-written message handler in the escape
hatch, or an `outbound` departure consumed by an `inbound` arrival that creates the row — a loopback
whose only function is to bypass the create-from's guard, at the cost of at-least-once delivery
semantics with no declared provenance and no relationship to the source visible in the model.

## Specification text

**Anchor:** Declarative glue > generates — create-from > event-driven creation — `event:`

An event-driven create-from binds to **either axis** of the event vocabulary: the source's own
lifecycle (`onTransition` / `onCreate`) or a **process step** — `onStepReached` /
`onStepCompleted: { process, step }`, the same binding notifications, integrations and departures
declare. A step-bound create-from expresses a follow-up document that belongs to a moment in a flow
rather than to a status write.

```yaml
generates:
  - name: log-activation
    from: Claim
    to: LogEntry
    event: { onStepCompleted: { process: ClaimApproval, step: activate }, mode: append }
    map: { Claim: id, amount: amount }
    defaults: { step: "activate", date: now }
```

> **Normative.** Exactly one trigger MUST be declared, from either axis. A step binding MUST name an
> existing process that declares a `trigger`, and an existing step that occupies a moment in the flow
> (a user task or a service task). The process's trigger entity MUST be the entity `from:` declares —
> a step event is about the record its process runs on, and that record is the one the create-from
> reads. A step binding whose source is owned by another model MUST be rejected: a process and its
> steps belong to the model that declares them. On the step axis the `when:` guard is optional; the
> step is the moment.

The `event` map MAY declare **`mode:`** — the cardinality of the trigger.

> **Normative.** `mode` is `once` or `append`; any other value MUST be rejected rather than read as
> the default. The default is `once`, which is the at-most-once behaviour above. Under `append` no
> existing-target lookup is performed and every delivered event MUST create another target row. The
> back-reference `map` entry is REQUIRED in both cardinalities — the dedup key under `once`, the
> created row's provenance under `append`. `mode` declared without a trigger MUST be rejected.

`mode: append` is the **absence** of a guard, not a state-aware one: event delivery is at-least-once,
so a redelivery appends a duplicate row, and a replacement for a target that was voided is not what
this cardinality expresses. Anything that must exist at most once per source keeps `mode: once`.

## DSL index

| Construct | What it does |
| --- | --- |
| [`generates.event.mode`](#event-driven-creation--event) | one target per source (`once`, default) or one per delivered event (`append`) |
