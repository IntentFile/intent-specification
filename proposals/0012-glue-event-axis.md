# The glue event axis - process-step events and inbound message/file arrivals


- **Status:** released in [1.4](../versions/1.4.md)
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/6537
- **Discussion:** https://github.com/IntentFile/intent-specification/pull/21

## Problem

A reacting glue entry (`notifications`, `integrations`) could bind only to an **entity lifecycle** event, and `inbound` could only be an HTTP webhook. Two things every real application needs were therefore inexpressible:

1. **A moment inside a process** — "when this task becomes available, tell the assignee's manager", "when that step completes, call the partner system". Only entity-level changes were observable; the process itself was event-silent.
2. **An arrival that is not HTTP** — a record that comes in on a queue or topic, or is dropped into a folder as a file.

## Proposed shape

Both extend an existing axis rather than adding a vocabulary.

```yaml
notifications:
  - name: reviewPending
    event: { onStepReached: { process: LoanApproval, step: librarianReview } }
    to: member.branch.managerEmail
    subject: "Loan {id} is waiting for review"
    body: "A librarian must approve it."

integrations:
  - name: pushActivation
    event: { onStepCompleted: { process: LoanApproval, step: activate } }
    method: POST
    url: "@config:PARTNER_URL"

inbound:
  - { name: leadHook,  path: /webhooks/lead, create: Lead }
  - { name: leadQueue, source: { queue: leads.inbound }, create: Lead }
  - { name: leadFeed,  source: { topic: crm.leads }, create: Lead }
  - { name: leadDrop,  source: { folder: /data/inbox/leads, cron: "0 */5 * * * ?" }, create: Lead }
```

## Expected behaviour (normative, platform-neutral)

**Step events.** A step event is an event *about the record the process runs on* — the process's `trigger` entity — so every action parameter (recipient rule, `{placeholder}` interpolation, `when:` guard, forwarded body) resolves exactly as for a lifecycle event; no action needs to know which axis fired it. A conforming generator rejects a binding whose process or step is not declared, whose step is not one that occupies an observable moment (a task, not a decision/wait/end), or whose process has no `trigger` (there is then no record to be about). `onStepReached` is observable before the step's own work begins; `onStepCompleted` after it finished **and** after that step's writes are persisted, so an observer never sees a stale record. One moment publishes once, however many entries observe it; a jump back into an observed step re-fires its `onStepReached` observers.

**Inbound arrivals.** Exactly one arrival per entry: `path`, or a `source` naming exactly one of `queue` / `topic` / `folder`. All three save through the entity's ordinary write path — the arrival is a transport, not a second data path, so validations, translations and the create event behave identically. A folder is **polled**, not watched (hence the mandatory `cron`, which is an error on the other sources); a file holds one record or an array of them, is not read while still being written, and leaves the drop folder once read — ingested and rejected files kept apart — so nothing is ingested twice and a rejection stays inspectable.

Conversation-shaped transports (acknowledgements, retries with backoff, certificates) stay beyond the scope boundary, as they are today.

## Prior art

Handled today by hand-written listener/job code alongside the generated application: a process listener that fetches the record and mails, a queue consumer that parses and saves, a scheduled folder scan. Each is mechanical, and each re-implements the recipient/interpolation rules the format already defines.

## Specification text

The prose below is what a release folds into the next version document, at the anchors
given. It was written against `versions/1.2.md` and is carried here unchanged.

**Anchor:** Overview > The scope boundary

| **Protocol adaptation** — conversation-shaped integration with an external system: certificates, acknowledgments, retries and their backoff, batch and file transports | [`integrations`](#integrations--outbound-http) and [`inbound`](#inbound--arrivals-from-outside) are one-line call-outs by design; a protocol has state and failure semantics no declaration should pretend to carry | an integration route in the platform's integration technology, feeding the entity's ordinary write path |

**Anchor:** Declarative glue

- **Event** — an entity `onCreate` / `onUpdate` / `onDelete` (with an optional `when:` guard), a process step reached or completed, a schedule (`cron`), or an inbound arrival (a webhook, a message, a dropped file).

**Anchor:** Declarative glue > Glue is generated integration code

### The event axis — lifecycle events and process-step events

A glue entry that reacts (`notifications`, `integrations`) declares **exactly one** `event:`, on one of two axes:

| Axis | Shape | Fires when |
| --- | --- | --- |
| entity lifecycle | `{ onCreate\|onUpdate\|onDelete: <Entity> }` | a record of that entity is created / updated / deleted |
| process step | `{ onStepReached\|onStepCompleted: { process: <Process>, step: <step> } }` | a running process arrives at that step / has just finished it |

```yaml
processes:
  - name: LoanApproval
    trigger: { onCreate: Loan }
    steps:
      - { name: librarianReview, kind: userTask,    args: { assignee: librarian, next: activate } }
      - { name: activate,        kind: serviceTask, args: { setField: status, value: ACTIVE } }

notifications:
  # "when the review task becomes available, tell the member's branch manager"
  - name: reviewPending
    event: { onStepReached: { process: LoanApproval, step: librarianReview } }
    to: member.branch.managerEmail
    subject: "Loan {id} is waiting for review"
    body: "A librarian must approve it."

integrations:
  # "when the loan has been activated, tell the partner system"
  - name: pushActivation
    event: { onStepCompleted: { process: LoanApproval, step: activate } }
    method: POST
    url: "@config:PARTNER_URL"
```

> **Normative.**
> A step event is an event **about the record the process runs on** — the process's `trigger` entity. Every action parameter therefore resolves exactly as it does for a lifecycle event: the same recipient rule, the same `{placeholder}` interpolation, the same `when:` guard, the same forwarded body. A conforming generator MUST reject a step event whose process is not declared, whose step is not declared in that process, whose step is not a `userTask` or a `serviceTask` (no other step kind occupies an observable moment), or whose process declares no `trigger` (there is then no record for the event to be about). `onStepReached` MUST be observable before the step's own work begins, and `onStepCompleted` after it has finished and after any writes that step performs (a task's edits, a `setField`) are persisted — an observer of a completed step never sees a stale record. Any number of entries may observe the same step moment; the record is published once. A `then`/`else` jump back into an observed step re-enters it, so its `onStepReached` observers fire again.

**Anchor:** Declarative glue > notifications

Email on an event of the axis above.

**Anchor:** Declarative glue > notifications

    event: { onUpdate: Order }     # one event of the event axis

**Anchor:** Declarative glue > integrations — outbound HTTP

### inbound — arrivals from outside

**Anchor:** Declarative glue > inbound — arrivals from outside

Another system tells us — a JSON record shaped like the entity, ingested into it. What differs between the three forms is only **where the record arrives**; the action is the same `create`.

**Anchor:** Declarative glue > inbound — arrivals from outside

  # HTTP — an endpoint the other system posts to
  - { name: leadHook,  path: /webhooks/lead, create: Lead }
  # message — every record arriving on a queue (point-to-point) or a topic (broadcast)
  - { name: leadQueue, source: { queue: leads.inbound }, create: Lead }
  - { name: leadFeed,  source: { topic: crm.leads }, create: Lead }
  # file — every file dropped into a folder, polled on the cron
  - { name: leadDrop,  source: { folder: /data/inbox/leads, cron: "0 */5 * * * ?" }, create: Lead }

**Anchor:** Declarative glue > inbound — arrivals from outside

> **Normative.**
> An `inbound` entry declares **exactly one arrival**: a `path` (HTTP) or a `source` naming exactly one of `queue` / `topic` / `folder`; declaring both, neither, or two channels is an error. The ingested record MUST be saved through the entity's ordinary write path, so validations, translations and the create event behave exactly as for any other write — the arrival is a transport, not a second data path. A `folder` source is **polled**, not watched, so its `cron` is required (and is an error on the other sources); a file MUST hold either one record or an array of them, MUST NOT be read while it is still being written, and MUST leave the drop folder once read — successfully ingested and rejected files kept apart — so that no file is ever ingested twice and a rejected one stays inspectable. Conversation-shaped transports (acknowledgements, retries with backoff, certificates) are out of scope by design; see [the scope boundary](#the-scope-boundary).

**Anchor:** Appendix A: DSL index

| [the event axis](#the-event-axis--lifecycle-events-and-process-step-events) | what a reacting glue entry binds to: an entity lifecycle event or a process step reached / completed |
| [`notifications`](#notifications) | email on an event of the axis |

**Anchor:** Appendix A: DSL index

| [`inbound`](#inbound--arrivals-from-outside) | records arriving from outside: a webhook, a queue/topic message, a dropped file |
