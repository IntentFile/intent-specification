# A scheduled generation declares its natural key

- **Status:** released in [1.9](../versions/1.9.md)
- **Issue:** [eclipse-dirigible/dirigible#7070](https://github.com/eclipse-dirigible/dirigible/issues/7070) (the key), [#7106](https://github.com/eclipse-dirigible/dirigible/issues/7106) (the period of the run); edge rules from [#7134](https://github.com/eclipse-dirigible/dirigible/issues/7134) (a null key term) and [#7133](https://github.com/eclipse-dirigible/dirigible/issues/7133) (one transaction per source row)
- **Implementation:** [eclipse-dirigible/dirigible#7079](https://github.com/eclipse-dirigible/dirigible/pull/7079), [#7117](https://github.com/eclipse-dirigible/dirigible/pull/7117), [#7166](https://github.com/eclipse-dirigible/dirigible/pull/7166), [#7168](https://github.com/eclipse-dirigible/dirigible/pull/7168)

## The problem

`schedules[].generate` creates one target record per matching source row on every tick, and the
1.6 text says nothing about what a **second** tick does. A second run of the same tick is not an
error case; it is ordinary: a deploy that failed halfway and was replayed, the scheduler's misfire
recovery after a restart, an operator pressing *Run* on the monitoring page to see the job work. Each
of those runs the same query against the same standing rows and creates everything again.

Walked on a staging instance: `monthly-project-timesheets` (`entity: Project`, `to: ProjectTimesheet`,
`defaults: { Period: now }`, children `EmployeeTimesheet` and under it `EmployeeDayAllocation`), run
twice from the monitoring page:

```
run 1: ProjectTimesheet 3 "2026 September - QA Project", EmployeeTimesheet 3, 22 day rows, a Submit task
run 2: ProjectTimesheet 4 "2026 September - QA Project", EmployeeTimesheet 4, 22 more day rows, a second task
```

Two identical project-months; the employee has two timesheets for the same month; both would bill.
The same shape sits under every recurring invoice, payroll run and project-month schedule in the
fleet, and the format had no way to say "this row's output for this tick already exists".

The guard that exists elsewhere does not fit. An event-driven create-from is at-most-once through a
**back-reference** to the one source record it was triggered from. A schedule's source is a *standing*
row: the same `Project` matches the query every month, so a back-reference alone would generate the
first project-month and never another. What identifies one tick's output is a pair of values the
target stores - `[Project, period]` - and the recurring-template family cannot even name that pair: a
monthly rent bill or a quarterly retainer invoice generated from a template is a plain document with
a `date`, no period column, no back-reference to the template.

## The proposed shape

`unique:` on `schedules[].generate` - the **natural key** of a generated record: the target
properties whose values identify one tick's output, each of them a property this same block assigns.

```yaml
schedules:
  - name: monthly-project-timesheets
    cron: "0 0 2 1 * ?"
    entity: Project
    generate:
      to: ProjectTimesheet
      unique: [Project, period]           # one project-month per project per period
      map: { Project: id, Customer: Customer }
      defaults: { period: now }           # a month field: the current YYYY-MM
      children:
        - to: EmployeeTimesheet
          parent: ProjectTimesheet
          forEach: { entity: Employee, match: { Project: Project } }
```

An entry may instead be **the period of the run** - `{ run: <period> }` - for a target that has no
period column to name:

```yaml
schedules:
  - name: monthly-recurring-bills
    cron: "0 0 5 1 * ?"
    entity: BillTemplate
    generate:
      to: PurchaseInvoice
      unique: [Supplier, supplierNumber, { run: month }]   # one bill per template per calendar month
      map: { Supplier: Supplier }
      defaults: { date: now, supplierNumber: "RECURRING - awaiting invoice" }
```

`run:` is `day`, `week`, `month`, `quarter` or `year`. It stores nothing: the guard ranges over the
`date` property this block assigns from `now`. When the block assigns more than one such date,
`{ run: month, of: date }` names the one the period is read from.

## Expected behaviour

**What the guard compares.** Before anything is built for a source row, a conforming generator looks
the target up by the key's terms, and the values it looks up are **the values the target WILL store**
for that row - rendered from this same block's `map` / `defaults`, including the shape `now` takes in
the target field (today's date for a `date`, the current `YYYY-MM` for a `month`, `YYYY-Www` for a
`week`). Deriving the lookup value a second way is what lets the value queried and the value written
drift, and "the same month" is comparable only because both sides render it identically. A hit means
this tick's output already exists.

**The skip is recorded, not errored.** A hit skips the source row and counts it; it is not a failure
and it does not stop the tick. The tick's summary carries the count alongside the rows created and the
rows that failed - `created [c] of [n] matching row(s), already existed [e], failed [f]` - because a
re-run that correctly did nothing is otherwise indistinguishable from a tick that never fired.

**Children go with their header.** The guard runs before the target header is constructed and skips
the **source row**, so the `children` fan-out under that header is skipped with it. A header-only guard
would find the existing header and still hang a second set of children under the first one - the half
of the duplicate that bills.

**A run term is a range over the document's own date.** `{ run: month }` on a block that writes
`date: now` is a lookup for a target with the other key terms whose date falls between the first and
the last day of the current month; `week`, `quarter` and `year` range likewise, `day` is equality on
today. A re-run on the 14th therefore finds what the 1st created, which is what makes *Run now* safe
on any day of the period. No hidden period column and no run ledger are added to the target - the
document already carries the period - and the shape works when the target is owned by another model,
where a column could not be added at all.

**One transaction per source row, fail-soft per row.** A source row's whole generation - the header,
its `children` and their children - is one unit of work: a row refused halfway (a missing required
value on the third child) leaves **nothing** behind. This matters *because of* the key: with the
header committed and its children not, every later tick would find the header, report it as already
existing and skip the row, so the partial result would be permanent. A row that fails is caught,
counted as `failed`, logged naming the source row's own key, and the rows after it are still
generated; the summary still logs.

**A null key term is null-safe and named.** The parse-time rule proves every key term is *assigned*,
not that its value is non-null at run time: a term mapped from a nullable field, or from a
`relation.field` off a null foreign key, binds null. The lookup treats a null term as "is null", so
the guard still finds this tick's own earlier output instead of matching nothing and duplicating on
every re-run. What a null term cannot do is tell two source rows apart: under the declared key two
rows that are both null there are one output, and the second is skipped as already existing. The tick
logs a warning naming the null term(s) and the source row when it happens; the fix is in the key.

**Absent, the key is advised, not required.** A scheduled generate with no `unique:` keeps generating
exactly what it did - every file authored before this proposal is unaffected - and the generation
reports an advisory saying that a second run will create another target per matching row. The one
exception already in the format: a schedule that declares `notify` *and* `generate` together requires
the key, because the key is what gates the mail.

**Best-effort against concurrency.** The guard is read-then-create, the same shape as the
event-driven create-from's `mode: once`: it does not defend two ticks running at the same instant. A
`unique:` business key on the target over the same columns remains the durable backstop.

## Edge rules

- **Every property term MUST be assigned by this block's `map` or `defaults`.** The guard queries the
  target by the values it is about to write, so a key column nothing sets is queried as null and can
  only match everything (nothing is ever generated again) or nothing (the duplicate this exists to
  stop); both are silent at run time, so it is an authoring error reported at generation, naming the
  schedule and the term. A property the `escalate:` ladder writes (`into:`) counts as assigned.
- A term that is **not a field or to-one relation of the target** MUST be rejected, where the target's
  model is at hand (a same-model target; for a target reached through `uses:` the check is not
  resolvable and the assignment rule alone applies).
- A **repeated** term, a **blank** entry, an entry naming **both** a property and a `run:` period, and
  **two** `run:` terms (one tick fires in exactly one period) MUST each be rejected.
- `of:` belongs to a `run:` term only; a property term declaring `of:` MUST be rejected - a property is
  compared to its own assigned value.
- **`run:` outside `day | week | month | quarter | year`** MUST be rejected naming the admitted set.
- **A `run:` term needs a date the run writes.** The date it ranges over MUST be a `date` property this
  block assigns from `now`. A block that assigns none MUST be rejected (there is nothing to compare;
  the message says to add `defaults: { date: now }`); a block that assigns more than one MUST be
  rejected until `of: <property>` names which - a due date a month out would key the bill into the
  next month. An `of:` naming a property the block does not assign from `now` MUST be rejected. A
  `month` / `week` typed field is *not* a candidate: it already holds the period as a value and is
  keyed on as an ordinary property term.
- **A key that is only a period MUST be rejected.** `unique: [{ run: month }]` identifies one target
  per period for the whole schedule, so the first matching row would generate and every other row be
  skipped as if it had already run - the silent inverse of the duplicate the key exists to stop. At
  least one property term stands beside it.
- **`unique:` on an on-demand `generates` MUST be rejected.** That action's cardinality is its event
  `mode` (`once`, guarded by the back-reference; `append`, unguarded); two differently shaped guards
  would leave two answers to "may this run again". The string shorthand and the `{ run: }` object form
  are one list: the shorthand `Project` means `{ property: Project }`.
- Choose the terms that identify the **run**, not the source. `[Project, period]`, never `[Project]`
  alone: a schedule's source is a standing row, so a back-reference-only key generates the first
  period and never another.
- Key on terms that are always assigned and never null; the null-safety above keeps a re-run a no-op,
  it does not make a null a discriminating value.
- **Disagreement with the 1.6 text.** The `schedules` section's `generate` example (`monthlyTimesheets`
  with `map: { Employee: id }`, `defaults: { Period: now }`) declares no key. The reference
  implementation now reports that shape as an advisory ("a second run will create another
  `EmployeeTimesheet` per matching `Employee`"), so the example is out of date rather than wrong; the
  Specification text below adds `unique: [Employee, Period]` to it. Nothing in 1.6 asserts a scheduled
  generate is idempotent, so no sentence is contradicted - the behaviour was simply unspecified, and
  unspecified read as "creates again".

## Prior art / workarounds

Before the key, the workarounds were procedural: never press *Run* on a generate schedule, never let
the scheduler recover a misfire, and delete the duplicate project-month (and its children, and the
inbox task the process opened for it) by hand when a replay happened anyway. The format's other
idempotency guards do not transfer - `generates.event` `mode: once` keys on a back-reference to one
source record and a schedule's source recurs every period; `posts.idempotentBy` and
`postings.backReference` are the same back-reference shape; CSV seeds upsert by their declared keys
but describe data, not a recurring generation.

The reference implementation shipped the property key in #7079 (issue #7070), the period-of-the-run
term in #7117 (issue #7106, which had proposed a hidden `<schedule>_period` column or a run ledger,
neither of which was built because the document's own date already holds the period), the null-safe
lookup with the named null term in #7166 (issue #7134), and the per-row unit of work with per-row
fail-soft in #7168 (issue #7133 - the interaction that made a partial tick permanent). Fleet adoption
followed in the modules' own schedules (`base-timesheets`, `base-purchase-invoices`,
`base-sales-invoices`).

## Specification text

**Anchor:** Declarative glue > schedules - appended after the paragraph that explains `map` / `defaults`
and `now` on the `generate` variant, before "Relative moments"; the `monthlyTimesheets` example in that
section gains the line `unique: [Employee, Period]           # the natural key - a re-run is a no-op`.

#### The natural key - `generate.unique`

A tick creates once per matching row, and a second run of the same tick is ordinary - a replayed
deploy, the scheduler's misfire recovery, an operator pressing *Run*. `unique:` names the **target**
properties whose values identify one tick's output, so the second run finds the first run's record and
creates nothing:

```yaml
schedules:
  - name: monthly-project-timesheets
    cron: "0 0 2 1 * ?"
    entity: Project
    generate:
      to: ProjectTimesheet
      unique: [Project, period]                 # one project-month per project per period
      map: { Project: id }
      defaults: { period: now }
```

Each term is a field or to-one relation of the target that this same block assigns through `map` or
`defaults`. Pick the terms that identify the *run*, not the source: a schedule's source is a standing
row, so `[Project]` alone would generate the first period and never another.

A target with no period column - a recurring bill or retainer invoice generated from a template, a
plain document with a `date` - keys on **the period of the run** instead:

```yaml
    generate:
      to: PurchaseInvoice
      unique: [Supplier, supplierNumber, { run: month }]   # one bill per template per calendar month
      map: { Supplier: Supplier }
      defaults: { date: now, supplierNumber: "RECURRING - awaiting invoice" }
```

`run:` is `day`, `week`, `month`, `quarter` or `year`. It stores nothing: the guard ranges over the
`date` this block assigns from `now` (`of: <property>` names it when the block assigns more than one),
so a re-run on any day of the period finds what its first day created.

> **Normative.**
> Before a source row's target is constructed, a conforming generator MUST look the target up by the
> key's terms, binding for each property term **the value the target will store** for that row -
> rendered from this block's `map` / `defaults`, in the target field's own shape - and for a `run:`
> term the range of the current period over the named date (`day` is equality on today). A hit MUST
> skip the whole source row, `children` included; the skip MUST be counted and reported in the tick's
> summary as already existing, alongside the rows created and the rows that failed, and MUST NOT be
> reported as an error. A null property value MUST be compared null-safely (as "is null"), so the guard
> still finds the row's own earlier output; when any term binds null the tick MUST log a warning naming
> the term(s) and the source row, since two rows sharing that null collapse to one output under the key.
> A source row's whole generation - the header, its `children` and theirs - MUST be one unit of work,
> so a row refused halfway leaves nothing for the guard to find later; a row's failure MUST be caught,
> counted, logged with the row's key, and MUST NOT stop the rows after it. The guard is read-then-create
> and need not defend two concurrent ticks; a `unique` business key on the target over the same columns
> remains the durable guarantee.
>
> Each of the following MUST be rejected at generation, naming the schedule and the term: a property
> term this block does not assign through `map` or `defaults` (a term the `escalate` ladder writes
> counts as assigned); a term that is not a field or to-one relation of a target whose model is at
> hand; a repeated term; a blank entry; an entry naming both a property and a `run:` period; a second
> `run:` term; `of:` on a property term; a `run:` value outside `day`, `week`, `month`, `quarter`,
> `year`; a `run:` term on a block that assigns no `date` property from `now`, or that assigns more
> than one without `of:` naming it, or whose `of:` names a property not so assigned (a `month` / `week`
> typed field is not a candidate - it holds the period and is keyed on as a property term); and a key
> whose only term is a `run:` period, which would identify one target per period for the whole
> schedule. `unique:` on an on-demand `generates` MUST be rejected: that action's cardinality is its
> event `mode`. A scheduled `generate` that declares no `unique:` MUST keep generating exactly as
> before, and the generation MUST report an advisory that a second run creates another target per
> matching row; a schedule declaring both `notify` and `generate` MUST require it.

## DSL index

| Construct | What it does |
| --- | --- |
| [`schedules[].generate.unique`](#the-natural-key---generateunique) | the natural key of a generated record - the target properties, as they will be stored, that make a re-run of the tick a recorded skip instead of a duplicate, children included |
| [`schedules[].generate.unique[].run`](#the-natural-key---generateunique) | key on the period of the run (`day` / `week` / `month` / `quarter` / `year`) as a range over the date the run writes, for a target with no period column; `of:` names the date when the block writes more than one |
