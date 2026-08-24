# Period locking - records dated in a closed period become read-only

- **Status:** accepted - awaiting the reference implementation
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/6535

## The problem

`immutableWhen:` freezes a record for what it **is**: a journal entry stops being editable because
its own status says POSTED. Accounting also needs the other axis - freezing a record for **when it
falls**. Once the accountant closes March, nothing dated in March may be created, edited or deleted
any more, whatever status the individual record carries: not a draft nobody posted, not a document
someone was still correcting, not a line added to an already-issued invoice. The close is the point
at which the month's figures were reported, and they must stop moving.

Nothing in the model expresses that today. The workaround is a status guard on every entity plus a
human process ("do not touch last month"), which is exactly the kind of rule that holds until the
first person who did not know about it.

## The proposed shape

Two declarations, because the two facts live in two places.

The **register** - an ordinary entity, whose rows are the dated windows - says which of its fields
are the bounds and which statuses mean closed:

```yaml
entities:
  - name: AccountingPeriod
    period:
      start: startDate
      end: endDate                       # inclusive
      closedWhen: "Status == CLOSED"     # the immutableWhen grammar; seeded names or ids
    fields:
      - { name: id,        type: integer, primaryKey: true, generated: true }
      - { name: name,      type: string, length: 100 }
      - { name: startDate, type: date, required: true }
      - { name: endDate,   type: date, required: true }
    relations:
      - { name: Status, kind: manyToOne, to: PeriodStatus, function: EntityStatus, init: OPEN }
```

Each **guarded** entity spends one line naming the register and which of its own dates decides which
window it falls in:

```yaml
  - name: JournalEntry
    immutableInPeriod: { period: AccountingPeriod, date: entryDate }
```

The date that counts is a modelling decision, not a convention - the issue date, the tax event date
and the posting date are three different answers and different jurisdictions pick different ones -
so it is authored, never inferred.

## Expected behaviour

While the register row covering a record's declared date is closed, the **user** write surface
refuses that record with a conflict response (the same one `immutableWhen` produces):

- **update** and **delete** of a record whose stored date falls in a closed window;
- **create** of a record dated inside a closed window - unlike the status guard, which has no create
  to protect (a fresh record has no status yet), refusing the create is most of what closing a
  period means;
- **update that moves** a record into a closed window, checked against the incoming date as well as
  the stored one - otherwise the guard would protect only what was already there.

Writes performed by the application itself - a workflow step, a roll-up, a generated reversal -
stay possible, exactly as for `immutableWhen`. A correction to a closed period is a reversal booked
in an open one, never an edit of the original, and the flow that books it must be able to write.

The pre-check a conforming generator exposes for status immutability answers for this guard too, so
a form opened on a frozen record renders read-only rather than failing on save.

Closing a period is a plain status transition on the register: a transition button, a lifecycle
edge, a workflow step. Nothing in this construct writes the register.

## Edge rules

- **A date covered by no period is open.** Periods are opened as they are needed, and an undeclared
  future month must not freeze what is booked into it, so "no covering row" can only mean open. A
  record whose date is unset falls in no period and stays writable.
- **Both bounds are `date` fields and both are required.** A timestamp bound would make "the period
  covering this date" depend on a time of day nobody authored. The end bound is inclusive.
- **`closedWhen` is the `immutableWhen` grammar** over the register's own `function: EntityStatus`
  relation: terms `<Status> == <seed id>` joined with `||`, seeded status names accepted. A register
  without such a relation, or without `closedWhen`, is a validation error - nothing would ever close
  it.
- **More than one covering row** is not an error: if any covering row is closed, the record is
  frozen. Overlapping periods are a data question, not a modelling one.
- **The guarded `date` must be a `date` field of the guarded entity**, matching the bounds it is
  compared against.
- **The lock reaches a composition child** of a guarded master, as a status lock does: a line write
  recomputes the document's totals, so a document dated in a closed period freezes its lines with
  it. The same per-child opt-out applies.
- **`immutableInPeriod` and `immutableWhen` compose.** An entity may declare either, both or
  neither; a record frozen by either is frozen.
- **The register is an entity of the same model.** The guard is generated alongside the entities it
  guards and reads the register directly; a register owned by another model is refused rather than
  generating a guard that silently never fires. Lifting this is follow-up work.

## Prior art / workarounds

Every accounting package has period close; none of them derives it - the periods are a register the
user maintains, and the lock is a rule the application enforces. Modelled today by hand: a status
field per document plus application code in each write path, or a database trigger, both of which
sit below the model and are invisible to the reader of the intent file.

Related constructs in this specification: `immutableWhen:` / `immutable:` (the status axis of the
same guard) and `resolves:` (the other effective-dated construct - a register whose covering row
supplies a value rather than a veto).
