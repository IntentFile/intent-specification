# A notify delivery is recorded on the record, and a failed one is an event

- **Status:** released in [1.9](../versions/1.9.md)
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7023
- **Implementation:** [eclipse-dirigible/dirigible#7033](https://github.com/eclipse-dirigible/dirigible/pull/7033),
  completed by [#7313](https://github.com/eclipse-dirigible/dirigible/pull/7313)
  (issue [#7290](https://github.com/eclipse-dirigible/dirigible/issues/7290): a failure before the send
  stamps too) and [#7378](https://github.com/eclipse-dirigible/dirigible/pull/7378)
  (issue [#7369](https://github.com/eclipse-dirigible/dirigible/issues/7369): no recipient stamps `skipped`)

## The problem

The notify block is fail-soft by design, and the 1.6 text says so: a `transitions[].notify` "MUST NOT
be able to fail its transition", a fan-out is "fail-soft per row at every call site", a recipient that
resolves to nothing is "skipped and recorded", and a failure "is recorded". The fail-soft half is right -
the status flip is the transition's contract, a schedule tick is a batch, and neither may break because a
mailbox is unreachable. The "recorded" half names no place. Nothing in the format says **where** the
recording lives, so a conforming generator satisfies the sentence with a server log line - and that is
what the reference implementation did.

The consequence, walked in a QA pass of a sales-invoice application: the user presses **Send**, the
invoice flips to SENT with its send method recorded, the button reports success, the mail never leaves
the server. Nothing is stamped on the invoice, nobody is told, and no construct in the format can react.
The customer reports the missing invoice weeks later, and in the meantime the dunning schedule has been
mailing reminders through the same broken sender, equally silently. "Which of last night's 400
reminders did not go out" is a question the application cannot answer.

The gap has three faces. The **record** carries no trace of its own delivery. The **acting person** sees
a green "done" for a message that was never handed over. And **glue** has nothing to bind: a lifecycle
event fires on the write, not on the send that followed it, so "open a task when the invoice mail
bounces" cannot be written.

## The proposed shape

One key on the notify block, at every call site, and one new kind on the event axis:

```yaml
entities:
  - name: SalesInvoice
    fields:
      - { name: id, type: integer, primaryKey: true, generated: true }
      - { name: number, type: string }
      - { name: sendOutcome, type: string, length: 128, readOnly: true }   # the trace's home
    relations:
      - { name: Customer, entity: Customer }
      - { name: Status, entity: SalesInvoiceStatus, function: EntityStatus, init: DRAFT }

transitions:
  - name: SendInvoice
    forEntity: SalesInvoice
    from: [DRAFT]
    setStatus: SENT
    notify:
      to: Customer.email
      subject: "Invoice {number}"
      body: "Please find your invoice attached."
      attach: print
      outcome: sendOutcome          # a string field of THIS record, length >= 64

processes:
  - name: ChaseDelivery             # "when the invoice mail bounces, someone chases it"
    trigger: { onNotifyFailed: SalesInvoice }
    steps:
      - { name: chase, kind: userTask, args: { assignee: billing, setRelationField: Status, value: SEND_FAILED, next: done } }
      - { name: done, kind: end }

notifications:
  - name: tellOpsAboutABounce       # or tell operations, with the reason in the body
    event: { onNotifyFailed: SalesInvoice }
    to: "@config:OPS_EMAIL"
    subject: "Invoice {number} could not be mailed"
    body: "The mail server said: {sendOutcome}"
```

`outcome:` names a string field of **the record the message is about** - the same record the block's
bare paths resolve against: the event record of a `notifications[]` entry, each matched row of a
`schedules[].notify`, the transitioned record of a `transitions[].notify`, the trigger record of a
`serviceTask`'s `args.notify`, and inside a `forEach` fan-out the **row**, since the row is what carries
the recipient. Every attempt stamps it. A failed attempt additionally publishes an event of a new kind,
`onNotifyFailed`, that binds wherever the event axis binds - a `notifications[]` or `integrations[]`
entry, an `outbound[]` departure, and a process `trigger:`.

**Resend is deliberately not a key.** Route the failure to a status of its own (`SEND_FAILED` above) and
declare the way back as an ordinary `transitions[]` button carrying the same notify block. The retry then
inherits the same guards, the same audit trail and the same outcome stamp; a bespoke resend affordance
would inherit none of them.

## Expected behaviour

**Three recorded states, one vocabulary.** The outcome field is stamped on every processed record, with
exactly one of:

| stamp | means |
| --- | --- |
| `sent` | the message was handed to the delivery channel |
| `failed: <reason>` | it was not - the reason is the channel's own message, after the literal prefix `failed: ` |
| `skipped` | there was nobody to send to: the recipient resolved to no address |

A processed record therefore never carries an empty outcome. An empty field means the block has not run
for this record; `skipped` means it ran and found no address - a data problem (a customer with no e-mail)
someone must see, and a list filter on the outcome is where they see it. `skipped` is not a failure and
publishes no event.

**Every failure of the attempt stamps, not only the send.** The attempt begins when the block starts
resolving what it needs - loading the related record the recipient path hops through, rendering the
attached document or report - and ends when the message is handed over. A failure anywhere in that span
is `failed: <reason>`. A block that fails while rendering a broken print template stamps exactly as one
whose mail server is down; otherwise the record stays empty precisely in the case the field exists to
surface.

**The stamp is a targeted, fail-soft write.** It writes the outcome column only, so it re-fires no
`onUpdate` reaction and cannot revert an edit that landed concurrently. It is not a user write: a record
made immutable by the very status the send followed (`immutableWhen`, a lifecycle lock) still records why
its own send failed. The reason is truncated to the field's declared length by the generator, so the
trace never truncates at the database where nothing reports what was cut. And the stamp itself never
fails the activity: a record *of* an outcome must not become a second, louder failure, and on the
per-row paths it would abort the rows still to send. A stamp that cannot be written is reported by the
implementation and the send goes on.

**Fail-soft is unchanged; silence is gone.** Whether an activity fails on a delivery failure is exactly
what the 1.6 text already fixes per call site - a transition never, a fan-out never per row, a sending
process step fails so the platform's retry applies. `outcome:` adds observation, not new failure paths.
Where an attempt is redelivered, each attempt stamps again, so the field always shows the latest.

**The acting person is told.** A transition that carries a notify block answers the delivery outcome
together with the record, and a generated surface reports `failed` as a **warning** naming the reason -
never a plain success for a message that did not leave. `skipped` is reported as its own state, because
reporting "nobody to mail" as a failure trains people to ignore the warning that matters.

**A failure is an event.** A `failed` stamp publishes an `onNotifyFailed` event about the record. The
event is published **with the stamp's own write**, so the trace and its announcement commit together: a
reaction never observes a record whose outcome field does not yet say what the event says. Only the
failure has a channel: a delivery that worked is the normal path, and announcing it would give every
reaction a second copy of an event it already has.

**The event's payload is the record**, as it stands after the stamp - so `{sendOutcome}` in a reacting
block reads the reason, a forwarded body carries it, and a process started by it runs on the record
whose mail bounced. `onNotifyFailed` takes the same `when:` guard as every other kind and, on a
`trigger:`, the same rules: the entity gains a back-reference so the process starts at most once for
the record, and `businessKey` / `businessKeyStrategy` apply unchanged.

## Edge rules

- **Type and length.** The outcome field MUST be a plain `string` field. When it declares a `length`, that
  length MUST be at least **64** - `failed: ` plus something of the channel's message worth reading. A
  shorter one is refused at authoring with a message naming the declared length and the minimum. A field
  with no declared length is accepted (the implementation's default string length applies, and the
  generator truncates to it).
- **What is refused, at authoring, with a message naming the block and the field:** a name that is **not
  a field** of the record the message is about; a name that is a **relation** of it (a status the failure
  should route to is what `onNotifyFailed` is for, and two writers of one status column is the collision
  the layer prevents - the message says so and names the alternative); a field of a **type other than
  string**, naming the type it was; and the record's **primary key**.
- **The field belongs to the record the message is about**, never to an anchor, a parent or a related
  entity - inside a fan-out that is the row, and the anchor record has no outcome of the rows' deliveries.
  A field of another entity is simply "not a field of" the record and is refused as such.
- **A cross-model source cannot carry an outcome.** A schedule over an entity owned by another model
  (`model: <alias>`) MUST refuse `outcome:`: the stamp writes through that record's own data layer and
  announces on its own failure topic, both of which belong to the owning model. Record the attempt where
  the record lives.
- **A `generate` schedule records nothing.** `outcome:` is a key of the notify block; a schedule with no
  `notify` has no delivery to record.
- **`onNotifyFailed` binds exactly one moment**, like every other kind: an `event:` or `trigger:` naming
  it together with another kind is refused with the usual "at most one of" message. The named entity
  MUST be declared. Binding it to an entity no notify block stamps is accepted - the reaction simply never
  fires - exactly as an `onDelete` on an entity nothing deletes is accepted.
- **The outcome field SHOULD be `readOnly`.** The stamp is a system write and a person's edit of the
  trace is worth nothing; a user-editable outcome also means a `when:` guard over it can be satisfied by
  hand. The format does not force this, because the field is the author's.
- **Where the 1.6 text and the shipped behaviour disagree:**
  - 1.6 says a block with no recipient is "skipped and recorded" and that a transition's delivery failure
    "is recorded". Without `outcome:` a conforming implementation has nowhere to record either, and the
    reference implementation records both as a server log line only - the state this proposal exists to
    remove. The Specification text below keeps the fail-soft sentences and replaces "recorded" with the
    declared home, so that the word means the same thing in both places.
  - 1.6 says the fan-out's per-row failure "is recorded" and the activity "completes with a per-row
    summary". The summary is a log line; the per-row record is `outcome:`, stamped on each row.
  - The `outbound` section leans on "the rule the notify block already sets" for a delivery failure that
    "MUST be recorded and MUST NOT fail the write". That rule now has a home for the notify block; the
    outbound text is not touched here.

## Prior art / workarounds

Before the construct: a hand-written listener around the mail client that writes a status column,
duplicated per call site and per module, and a resend button of its own with none of the transition's
guards. The reference implementation's `resolves:` already stamps a lookup's `outcome:` on the record it
tried to fill with `found` / `notFound` / `ambiguous`, so unresolved records form a filterable worklist;
this proposal applies the same shape to a delivery. A generated `<Entity>NotificationLog` child was
considered and set aside: the question the business asks is "did THIS document's mail go out", which is a
column of the document, not a join.

## Specification text

**Anchor:** Declarative glue > The notify block — and `attach: print`, sending the document itself.
The second `> **Normative.**` blockquote of that section (the fail-soft rule: "A block whose recipient
resolves to no address is a **no-op** ...") is REPLACED by the blockquote below, and the prose and the
remaining blockquotes follow it as a new sub-heading `#### Recording the delivery — outcome:` placed
before *Links back to the application*. The event-axis table in *The event axis — lifecycle events and
process-step events* gains the row given at the end, and the `trigger` list in *Processes & forms >
trigger* gains `onNotifyFailed` among the kinds.

> **Normative.** A block whose recipient resolves to no address is a **no-op**: the send is skipped and,
> when the block declares an `outcome`, the record is stamped `skipped` - never an error, because a record
> with nobody to notify must not stall a flow. A `transitions[].notify` MUST NOT be able to fail its
> transition: the status flip is the transition's contract and is already applied when the message is
> attempted, so a delivery failure is stamped on the record (when an `outcome` is declared), reported to
> the acting person as a warning, and the transition still reports success. At the other call sites a
> delivery failure MAY fail the activity so the platform's own retry applies; a sending process step
> SHOULD, since the message is that step's whole purpose. Whether or not the activity fails, the outcome
> is stamped.

#### Recording the delivery — `outcome:`

A notify block is fail-soft everywhere. Until it names an outcome field it is also silent: the write
commits, the message never leaves, and the only trace is wherever the implementation logs. `outcome:`
gives the delivery a home on the record:

```yaml
    notify:
      to: Customer.email
      subject: "Invoice {number}"
      body: "Please find your invoice attached."
      attach: print
      outcome: sendOutcome          # a string field of the record the message is about, length >= 64
```

The field belongs to **the record the message is about** - the record every bare path of the block
resolves against; inside a [fan-out](#one-message-per-related-row-foreach), the row. Every attempt stamps
it with one of three values:

| stamp | means |
| --- | --- |
| `sent` | the message was handed to the delivery channel |
| `failed: <reason>` | it was not; the channel's own message follows the literal prefix `failed: ` |
| `skipped` | the recipient resolved to no address |

> **Normative.** `outcome` MUST name a `string` field of the record the message is about, and that field
> MUST NOT be the record's primary key. A field of another entity, a relation, a field of another type
> and the primary key MUST each be refused at authoring with a message naming the block, the field and
> the reason; the refusal of a relation SHOULD name `onNotifyFailed` as the way to route a status. A
> declared `length` MUST be at least 64; a shorter one MUST be refused naming the declared length and the
> minimum. A `schedules[].notify` over a cross-model source (`model:`) MUST refuse `outcome`: the stamp
> and its event belong to the model that owns the record.

> **Normative.** A processed record MUST never carry an empty outcome: exactly one of `sent`,
> `failed: <reason>` and `skipped` MUST be stamped per attempt, with exactly that spelling and prefix, so
> a list filter over the field means the same thing in every application. The attempt spans everything
> the block does for the record - resolving the recipient, loading the related record a path hops
> through, rendering an attached document or report, and the hand-over itself - and a failure anywhere
> in that span MUST stamp `failed: <reason>`. A reason longer than the field MUST be truncated by the
> generator, never by the store.

> **Normative.** The stamp is a **targeted write of the outcome column only**: it MUST NOT publish the
> record's update event, MUST NOT overwrite any other column, and MUST NOT be refused by a user-write
> immutability the record has meanwhile acquired (`immutableWhen`, a lifecycle lock) - the record records
> its own delivery whatever state it is in. The stamp MUST NOT fail the activity that made it: a stamp
> that cannot be written is reported and the send, or the rows still to send, go on. On a call site that
> reports to the acting person, `failed` MUST be reported as a warning naming the reason and `skipped` as
> its own state, distinct from both `sent` and `failed`.

> **Normative.** A `failed` stamp MUST publish an **`onNotifyFailed`** event about the record, in the
> same write as the stamp, so a reaction never observes a record whose outcome disagrees with the event
> it received. `sent` and `skipped` MUST NOT publish it: only the failure has a channel. The event's
> payload is the record as it stands after the stamp, and it binds wherever the event axis binds - a
> `notifications[]` or `integrations[]` entry, an `outbound[]` departure and a process `trigger:` - with
> the same `when:` guard, the same one-moment rule and, on a trigger, the same back-reference and
> business-key rules as every other kind. Naming an entity no notify block stamps is accepted; the
> reaction never fires.

The format has no resend key. Route the failure to a status of its own with a process or a reaction on
`onNotifyFailed`, and declare the way back as an ordinary `transitions[]` button carrying the same notify
block: the retry inherits the guards, the audit trail and the stamp.

The event-axis table gains:

| Axis | Shape | Fires when |
| --- | --- | --- |
| delivery | `{ onNotifyFailed: <Entity> }` | a notify block about a record of that entity, declaring an `outcome`, stamped `failed` |

## DSL index

| Construct | What it does |
| --- | --- |
| [`notify.outcome`](#recording-the-delivery--outcome) | stamp `sent` / `failed: <reason>` / `skipped` on the record the message is about, in a declared string field, on every attempt - fail-soft, targeted, never empty for a processed record |
| [`onNotifyFailed`](#recording-the-delivery--outcome) | the delivery axis: an event about the record whose notify attempt failed, published with the stamp; binds in `notifications`, `integrations`, `outbound` and a process `trigger` |
