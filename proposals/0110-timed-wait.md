# A wait step may park for a duration, so a process polls by a loop, not by a retry

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/136, https://github.com/eclipse-dirigible/dirigible/issues/8040, https://github.com/eclipse-dirigible/dirigible/issues/8106
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8045, https://github.com/eclipse-dirigible/dirigible/pull/8451

## The problem

A process often waits for a state it **expects** to change: a remote export to finish, a resource to
be released, another record to settle. The format has a check (a task), a decision, and loops
through a wait. But its only wait parks until an **entity event** ("A `wait` step parks the process
until an entity lifecycle event resumes it"), and its only timers are a user task's `timeout` /
`expire` boundaries. There is no way to say "pause ten seconds, then look again".

Models poll through a step's retry instead: the task fails with "not yet", and
`retry: { count: 60, every: PT10S }` paces the next check. That makes every expected "not yet" a
failure:

- each check is reported as one;
- the attempt budget belongs to waiting rather than to real errors, so a genuinely failing check
  has no retries of its own left;
- an exhausted budget reads as a technical fault.

Retry is for unexpected failures.

A second gap sits in the event wait itself. 1.11 makes it **fail-soft**: "no parked instance, or an
instance already past the wait, is a no-op". The steps between a user task and the wait run on
their own, so an event can arrive **before** the process reaches the wait - a reply posted the
moment the agent completes the task that asked for it. That event found no parked instance, was
dropped as the no-op, and the instance then parked on a wait whose event had already happened.

## The proposed shape

```yaml
steps:
  - { name: checkExport, kind: serviceTask, args: { delegate: CheckExport, produces: [exportState], retry: { count: 3, every: PT30S }, onError: recordFailure, next: exportDone } }
  - { name: exportDone,  kind: decision,    args: { if: "exportState == 'DONE'", then: publish, else: pause } }
  - { name: pause,       kind: wait,        args: { after: PT10S, next: checkExport } }   # a timed wait
  - { name: publish,     kind: serviceTask, args: { delegate: Publish, next: done } }
```

The check keeps its `retry` for real failures; "not yet" is a decision branch into a timed wait
that loops back to the check.

## Expected behaviour

A `wait` step may declare **`after: <ISO-8601 duration>`** instead of a resuming event. The process
parks at the step for that duration, then continues at `next`. Looping back through a timed wait is
a cycle through a wait state, which is legal. A check answering "not yet" twice and then "done"
completes after three checks and two pauses, with no failed attempt recorded.

An **event** wait keeps an event that arrives early. When the instance is running but not parked on
the wait - the steps before it still running, or a timed wait before it still pending - the event
is kept, and the wait continues on it as soon as it is armed, instead of parking. Only an event that
arrived since the process left its previous user task counts: a kept event is discarded when a user
task completes, so a reply posted while the task that asks for it is still open does not answer the
wait that follows it, and the wait parks until a later one. A kept event is spent when the wait is
left, whatever resumed it. An instance that is no longer running is still the no-op.

## Edge rules

- A `wait` declares either `after`, or exactly one resuming event (with its `via` / `when`). `after`
  with an event, `after` with `via` or `when`, and neither form MUST be rejected, the report naming
  the step.
- `after` MUST be a positive ISO-8601 duration or period (`PT10S`, `PT5M`, `P1D`); a zero, a
  negative and a non-ISO value (`PT0S`, `PT-10S`, `10s`) are rejected.
- A timed wait correlates no record, so it does not require the process to declare a `trigger:`.
- Aborting the instance cancels its pending timed wait, as `abortOn` already cancels parked waits.
- An event wait without an early event behaves as before, and a process without an event wait is
  unaffected by the early-event rule. `abortOn` is unaffected: it listens from the start of the
  instance.

## Prior art / workarounds

Polling through a service task's retry cycle, with every "not yet" a failed attempt. For the early
event, nothing: the event was lost and the instance had to be moved by hand. Reference
implementation: https://github.com/eclipse-dirigible/dirigible/pull/8045 (the timed wait) and
https://github.com/eclipse-dirigible/dirigible/pull/8451 (the early event).

## Specification text

**Anchor:** Processes & forms > processes > wait - park the process on a data event. After the
bullet list, add the bullet and the paragraph below; the paragraph that follows the list is
amended as stated at the end.

- `after: <ISO-8601 duration>` - instead of a resuming event, a **timed wait**: the process parks
  for the duration, then continues at `next`.

```yaml
steps:
  - { name: exportDone, kind: decision, args: { if: "exportState == 'DONE'", then: publish, else: pause } }
  - { name: pause,      kind: wait,     args: { after: PT10S, next: checkExport } }
```

The timed wait is how a process **polls**: a task reads a state and produces it, a decision routes
on it, and a timed wait loops back to the task. A step's `retry` is for unexpected failures, not for
a state the process expects to wait for.

> **Normative.** A `wait` MUST declare either `after` or exactly one resuming event; `after` together
> with an event, `via` or `when`, and a wait declaring neither, MUST be rejected at authoring time,
> the report naming the step. `after` MUST be a positive ISO-8601 duration or period; a zero, a
> negative or a malformed value MUST be rejected. A timed wait correlates no record and MUST NOT
> require a `trigger:`. Cancelling or aborting the instance MUST cancel its pending timed wait.

The paragraph after the list is amended. "a `wait` requires the process to declare a `trigger:`"
becomes "an event `wait` requires the process to declare a `trigger:`", and "It is **fail-soft**: no
parked instance, or an instance already past the wait, is a no-op - never an error." becomes:

It is **fail-soft**: an instance that is no longer running is a no-op - never an error. An event
that arrives **before** the instance is parked on the wait is not lost: it is kept, and the wait
continues on it as soon as it is armed.

> **Normative.** An event matching an event wait that arrives while the instance is running but not
> parked on it MUST be kept, and the wait MUST continue on the kept event once it is armed rather than
> park. Only an event that arrived since the process left its previous user task counts: a kept event
> MUST be discarded when a user task of the instance completes. A kept event MUST be spent when the
> wait is left, whatever resumed it. An event for an instance that is no longer running MUST be the
> no-op.

## DSL index

| Construct | What it does |
| --- | --- |
| [`wait.after`](#wait--park-the-process-on-a-data-event) | a timed wait: park for a positive ISO-8601 duration, then continue at `next` - how a process polls, by a loop through a decision rather than by a retry; needs no `trigger:` |
