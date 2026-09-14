# A scheduled reminder records what it sent and escalates by days past due

- **Status:** draft
- **Issue:** [eclipse-dirigible/dirigible#7276](https://github.com/eclipse-dirigible/dirigible/issues/7276),
  [eclipse-dirigible/dirigible#7365](https://github.com/eclipse-dirigible/dirigible/issues/7365)
- **Implementation:** [eclipse-dirigible/dirigible#7342](https://github.com/eclipse-dirigible/dirigible/pull/7342),
  [eclipse-dirigible/dirigible#7376](https://github.com/eclipse-dirigible/dirigible/pull/7376)

## The problem

Dunning is the archetypal schedule, and a `schedules[]` entry cannot express it. The 1.6 text
allows **exactly one** of `notify` or `generate` per row, so a payment reminder is either a mail
that leaves no trace or a record that mails nobody. And nothing distinguishes a document three days
overdue from one ninety days overdue: the same wording goes out every week for as long as the row
matches.

A billing module that models dunning correctly at the data level shows both gaps at once. It has
`ReminderLevel`, a setting entity seeded First reminder (3 days), Second reminder (14), Final
notice (30); `PaymentReminder`, the history - a composition child of the invoice with a `Level`
relation - and a report over it; and a weekly `notify`-only schedule over overdue invoices. Yet:

- the history, and the report over it, fill only from a person's manual clicks, never from the
  automated sends - it is not a record of what went out;
- the second and final levels are seeded but **inert**: no construct picks a level from how overdue
  the row is, and no construct sends each level once, so only the first wording is ever applied.

The module's own comments defer the behaviour to "the outbound e-mail glue". The alternative is a
hand-written job that queries, picks, sends and logs - the escape hatch the declarative model exists
to avoid.

## The proposed shape

Two things, declared on the same `schedules[]` entry.

**`notify` and `generate` together.** A schedule may declare both: per matched row it writes the
target record and sends the message, as one unit. Because the same row matches on every tick, a
combined schedule MUST carry a `generate.unique:` key (proposal 0045) - the key gates both halves,
so a row whose record already exists is skipped entirely, mail included.

**`escalate:` - a ladder of levels chosen by days past a date.** The ladder is an ordinary entity of
the model (normally a setting entity), the threshold an integer field on it, and the date the days
are counted from a `date` field of the queried row. The chosen level is written onto the generated
record through a to-one relation, and that relation is part of the `unique:` key, so each level of
each row goes out once.

```yaml
entities:
  - name: ReminderLevel                      # the ladder, seeded First(3) / Second(14) / Final(30)
    kind: setting
    fields:
      - { name: id,           type: integer, primaryKey: true, generated: true }
      - { name: name,         type: string }
      - { name: daysAfterDue, type: integer }
      - { name: wording,      type: string, length: 500 }
  - name: PaymentReminder                    # the HISTORY - what was sent, and at which level
    fields:
      - { name: id,     type: integer, primaryKey: true, generated: true }
      - { name: sentOn, type: date }
    relations:
      - { name: SalesInvoice, kind: manyToOne, to: SalesInvoice, composition: true, required: true }
      - { name: Level,        kind: manyToOne, to: ReminderLevel }

schedules:
  - name: overdue-invoice-reminders
    cron: "0 0 8 * * MON"                    # every Monday at 08:00
    entity: SalesInvoice
    where:
      - { field: Status, op: eq, value: OVERDUE }
      - { field: dueOn,  op: lt, value: CURRENT_DATE }
    escalate:
      ladder: ReminderLevel                  # the entity holding the levels
      after: daysAfterDue                    # integer field of the ladder: the threshold in days
      since: dueOn                           # date field of the queried row the days are counted from
      into: Level                            # to-one relation of the generated target that receives the level
    generate:
      to: PaymentReminder
      unique: [SalesInvoice, Level]          # (document, level) - each level goes out ONCE
      map: { SalesInvoice: id }
      defaults: { sentOn: now }
    notify:
      to: contactEmail
      subject: "Invoice {number} - {escalation.name}"
      body: "{escalation.wording}"           # the chosen level's own text
      attach: print
```

The four keys of `escalate:` are all required. `{escalation.<field>}` in the subject or body reads
one field of the chosen level; it is the per-level wording the ladder exists for.

## Expected behaviour

Per matched row, on every tick, in this order:

1. **Level selection.** The days past due are the whole days between the row's `since` value and
   the day of the firing. The level applied is the **highest** ladder row whose `after` threshold
   the row has met or passed (`days >= after`). A ladder row with no threshold takes part in no
   selection.
2. **No matching level.** A row that has met **no** threshold - a document one day overdue against
   a ladder whose lowest rung is three, or a row whose `since` value is empty - is **left for a
   later tick**. It is not mailed at the bottom rung, no record is written, and the tick reports how
   many rows were in that state, so "matched 40, created 3" reads as the ladder working rather than
   as a query that nearly missed.
3. **The guard.** With a level chosen, the `unique:` key - which includes `into` - is checked
   against the existing target records. A record for this (row, level) already exists: the row is
   skipped entirely, mail included, and counted as already existing. This is what advances First to
   Second to Final as the document ages: the first level's record exists, the second level's does
   not until its threshold is met.
4. **The recipient**, resolved before anything is written. A row with no recipient is counted as
   such and leaves no record: the history never says a message went out to nobody.
5. **Record and send, as one unit.** The target record is created through the target's own write
   path - `map`, `defaults`, and `into` set to the chosen level - and the message is sent as the
   **last act of the same unit of work**. A delivery that fails rolls the record back: no record,
   the row is counted as failed, and the next tick finds an unsent row and completes it. The
   residual window is a commit that fails after a successful send, which costs a duplicate message
   rather than a lost one - the right way round for a reminder.

One bad row costs one failure; the tick continues with the next row. A combined tick reports one
summary - created-and-mailed, already existed, no recipient, failed - because created and mailed
are one number once they are one unit.

`{escalation.<field>}` resolves against the chosen level in both subject and body. Every other
placeholder of the notify block - bare fields, one-hop `relation.field`, the link tokens,
`attach: print` - keeps its meaning.

A schedule that declares only `notify`, or only `generate`, is unchanged by this proposal.

## Edge rules

- **Contradiction with the 1.6 text.** The `schedules` section reads "Exactly one of `notify` or
  `generate` per row." The reference implementation accepts both since #7342, and the Specification
  text below replaces that sentence. A combined schedule without `generate.unique:` is refused,
  naming the schedule and saying that without a key the tick would re-mail every matched row on
  every firing and write another record beside each send.
- **`escalate` requires `generate`.** Without a record nothing distinguishes a level already sent
  from one still due, so a `notify`-only escalation would re-send its top level every tick - the
  behaviour the ladder replaces. Refused, telling the author to add a generate whose `unique:` key
  names the `into` property.
- **`into` MUST be a term of `generate.unique:`.** Keyed on the document alone, the guard finds the
  first reminder of a row forever and no row is ever escalated - the exact symptom reported. Refused,
  telling the author to key on the row's back-reference AND `into`.
- **`into` MUST be a to-one relation of the generate target that points at the ladder.** A property
  that is not a to-one relation, or one whose target is another entity, is refused naming what it
  does point at.
- **`into` MUST NOT also be assigned by the generate's `map` or `defaults`** - two answers to which
  level applies. Refused, telling the author to remove the other assignment.
- **`after` MUST be an `integer` field of the ladder; `since` MUST be a `date` field of the queried
  entity.** The ladder counts whole days. A missing field is refused as not a field of the ladder /
  of the queried entity; a field of another type is refused naming the type it has.
- **`ladder` MUST be an entity of this model.** Refused otherwise.
- **The source and the generate target MUST both be local.** The days are counted off the queried
  row's own date, and whether the target's `into` points at this model's ladder is knowable only in
  the model that owns the target. A cross-model source (`model: <alias>`) or a cross-model target
  (`uses:` alias) is refused, telling the author to keep an escalating schedule in the model that
  owns the row it ages and to keep the history entity local. The cross-model source forms of a plain
  `notify` or `generate` schedule are untouched.
- **`{escalation.<field>}` is validated at authoring time.** On a schedule with no `escalate:`
  there is no level to read; a field the ladder does not declare, an empty field, or a walk on
  (`{escalation.Level.name}`) is not a field of the ladder. Both are refused - an unresolvable
  placeholder would otherwise degrade to its literal characters in the customer's mail. The
  placeholder reads one field of the level, never a path.
- A ladder MAY hold several rows with the same threshold; which of them is chosen is not specified.
  Give a ladder distinct thresholds.
- A ladder row whose threshold is empty is never selected; a queried row whose `since` value is
  empty is never escalated and counts as not yet due.
- The atomicity contract in step 5 holds for every combined schedule, escalating or not. It is what
  makes the `unique:` guard safe: a record exists only where the message went out, so skipping a
  row with a record never loses a delivery.
- `escalate` is not admitted on a `generate`-only schedule's `children`, on `notifications`, on a
  process step's `notify`, or on `generates` (create-from): the construct is about how overdue a
  **queried** row is, and only a schedule queries rows by age.

## Prior art / workarounds

A hand-written scheduled handler: query the overdue documents, load the ladder, pick the level,
send, write the history row - one more place that knows the level ids and the mail format, and the
exact escape hatch a module stays declarative to avoid. Or two schedules, one that mails and one
that records, which cannot agree on the level and cannot be made idempotent together. Or a manual
"send reminder" action whose history row defaults to the first level and asks a person to edit the
level by hand.

The reference implementation ships this shape in #7342, and #7376 moved the send inside the unit
that writes the record after a failed delivery had been found to leave a committed record that the
guard then honoured as sent.

## Specification text

**Anchor:** Declarative glue > schedules. The sentence "Exactly one of `notify` or `generate` per
row." in the section's opening paragraph is replaced by the first paragraph below; the remaining
text follows the `generate` example and precedes "Relative moments — a `where` value offset from
now".

A row's action is `notify`, `generate`, or both. A schedule that declares both writes the target
record and sends the message per matched row, as one unit; it MUST also declare a `generate.unique:`
key, which gates both halves - a row whose record already exists is skipped entirely, mail included.

#### escalate — a level chosen by days past a date

A reminder that repeats is rarely one wording: it is a first reminder, a second, a final notice, as
the document ages, each sent once. `escalate:` places the queried row on a **ladder** - an entity of
the model, normally a setting entity, whose rows are the levels - by how many whole days have passed
since a date of the row, writes the chosen level onto the generated record, and lets the message
read the level's own fields.

```yaml
schedules:
  - name: overdue-invoice-reminders
    cron: "0 0 8 * * MON"
    entity: SalesInvoice
    where:
      - { field: Status, op: eq, value: OVERDUE }
      - { field: dueOn,  op: lt, value: CURRENT_DATE }
    escalate:
      ladder: ReminderLevel        # the entity whose rows are the levels
      after: daysAfterDue          # integer field of the ladder: the threshold in days
      since: dueOn                 # date field of the queried row the days are counted from
      into: Level                  # to-one relation of the generate target that receives the level
    generate:
      to: PaymentReminder
      unique: [SalesInvoice, Level]   # the level is part of the key: each level goes out once
      map: { SalesInvoice: id }
      defaults: { sentOn: now }
    notify:
      to: contactEmail
      subject: "Invoice {number} - {escalation.name}"
      body: "{escalation.wording}"
      attach: print
```

The level applied is the **highest** ladder row whose `after` threshold the row has met
(`days since >= after`). A row that has met none is left for a later tick - not mailed at the
bottom rung - and the tick reports how many rows were in that state. Because `into` is part of the
`unique:` key, the guard finds the first level's record and not the second's, which is what advances
the row from level to level as it ages. `{escalation.<field>}` in the subject or body reads one
field of the chosen level.

> **Normative.**
> A schedule MAY declare both `notify` and `generate`. One that does MUST declare `generate.unique:`
> and MUST, per matched row, resolve the recipient before writing anything, create the target record
> and send the message inside **one unit of work**, the send last, so that a failed delivery leaves
> no record and the row is completed by a later tick; a row with no recipient MUST leave no record.
> The `unique:` key gates both halves: a row whose record exists MUST be skipped entirely. A
> combined tick MUST report one summary, counting failures per row.
>
> `escalate:` MUST declare all of `ladder`, `after`, `since` and `into`. `ladder` MUST be an entity
> of this model; `after` MUST be an `integer` field of the ladder; `since` MUST be a `date` field of
> the queried entity; `into` MUST be a to-one relation of the generate target whose target is the
> ladder, MUST be a term of `generate.unique:`, and MUST NOT also be assigned by the generate's
> `map` or `defaults`. `escalate` without a `generate` MUST be refused. An escalating schedule's
> queried entity and generate target MUST both be local: a cross-model source or target MUST be
> refused. Days are counted in whole days from the row's `since` value to the day of the firing; the
> level applied MUST be the one with the highest threshold the row has met; a ladder row with no
> threshold takes part in no selection; a row that has met no threshold, or whose `since` value is
> empty, MUST be neither written nor mailed on that tick and MUST be counted. `{escalation.<field>}`
> MUST name one declared field of the ladder - never a path - and MUST be refused on a schedule that
> declares no `escalate:`. Each of these refusals MUST be an authoring error naming the schedule,
> reported before generation.

## DSL index

| Construct | What it does |
| --- | --- |
| [`schedules.escalate`](#escalate--a-level-chosen-by-days-past-a-date) | place a queried row on a ladder of levels by whole days past one of its dates; write the level onto the generated record, key on it, and read its fields as `{escalation.<field>}` in the message |

The existing `schedules` row is amended from "cron: notify or generate records per matching row" to
"cron: notify and/or generate records per matching row".
