# Records generated from a report's rows - `fromReport`: the lines of one document, or one record per row, with `undo`

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8329
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8534 (items mode),
  https://github.com/eclipse-dirigible/dirigible/pull/8535 (rows mode)

**Scope.** This proposal has two modes over one report-reading rule. In **items mode**
(`items.fromReport`), a create-from's header is built from the source record as before, and its
lines are a report's result rows - a year close, undone by `undo: reverse`. In **rows mode**
(`fromReport` on the generation itself), there is no header and every result row is one record of
its own - next year's opening balances, undone by `undo: delete`. Both modes read three derived
closing-balance columns.

## The problem

A year close is generated from the ledger's own balances: one closing entry with a line per result
account, taking the account's year balance off it, and the opposite booked on the account it closes
into. Every one of those numbers is a row of a `kind: balance` report over the year's window. No
construct writes records from report rows: a create-from copies one record's header and its child
rows, a reversal copies one entry's lines, a posting books one entry per source event. So the close
can only be written by hand, together with its undo and its guard against a second run.

Next year's opening balances are the same numbers in another shape: one record per balance-sheet
account - and, where the account keeps a subledger, per counterparty and open document - carrying its
closing balance. There is no document: each row is a record of its own. Undoing them means deleting
exactly the records one run wrote, and a second run must not carry the same balances forward twice.
A settled document's closing debit and credit are equal, so it should carry nothing forward, which
the report's separate closing debit and closing credit columns cannot say on their own.

## The proposed shape

```yaml
generates:
  - name: closeYear
    from: FiscalYear
    to: Journal
    map: { Year: id, date: endDate }
    items:
      fromReport: YearBalance
      parameters: { fromDate: startDate, toDate: endDate }
      to: JournalLine
      where:
        - { field: Account.ClosingAccount, op: ne, value: null }
      map: { Account: Account.id, debit: closingCredit, credit: closingDebit }
      counter: { Account: Account.ClosingAccount, debit: closingDebit, credit: closingCredit }
    undo: reverse
```

- `items.fromReport` - a `kind: balance` report without `correspondence`. Its result rows are the
  lines, instead of the rows of `items.from`; the two are mutually exclusive.
- `items.parameters` - report parameter -> property of the source record. `fromDate` and `toDate`
  are required.
- `items.map` - line property -> report column. A column is a report dimension (`Account.id`) or one
  of the six balance amounts `openingDebit`, `openingCredit`, `debit`, `credit`, `closingDebit`,
  `closingCredit`, or one of three derived ones: `closingBalance` (closing debit minus closing
  credit), `closingNetDebit` and `closingNetCredit` (that difference on the side it stands on, zero
  on the other).
- `items.where` - optional `{ field, op, value }` conditions over report columns; `op` is
  `eq` / `ne` / `gt` / `ge` / `lt` / `le`; `value: null` with `eq` / `ne` tests for an empty cell.
- `items.counter` - optional: the cells of a counter line, in the shape of `items.map`.
- `items.defaults` - optional literals set on every line and counter line.
- `undo` - optional, `reverse`: an undo action on the source record.

Rows mode, one record per row:

```yaml
generates:
  - name: openingBalances
    from: FiscalYear
    to: OpeningBalance
    fromReport: YearBalance        # dimensions: [Account.id, Account.ClosingAccount, Partner.id, Document.id]
    parameters: { fromDate: startDate, toDate: endDate }
    where:
      - { field: Account.ClosingAccount, op: eq, value: null }
      - { field: closingBalance, op: ne, value: 0 }
    backReference: Year
    map: { Account: Account.id, Partner: Partner.id, Document: Document.id,
           debit: closingNetDebit, credit: closingNetCredit }
    undo: delete
```

- `fromReport` - on the generation itself: a `kind: balance` report without `correspondence`; each
  result row becomes one record of `to`. Mutually exclusive with `items`.
- `parameters`, `where` - as in items mode.
- `backReference` - required: the to-one relation of `to` pointing at `from`, written on every record.
- `map` - record property -> report column, as `items.map`; `defaults` - literals on every record.
- `undo` - optional, `delete`: an undo action on the source record.

## Expected behaviour

- The header is built from the source record as for any create-from, and its `map` copies the
  source's key onto the target.
- The report is run with each `parameters` entry read off the source record. A source record with no
  value for a bound property refuses the run instead of reading the report's default window.
- Each result row satisfying every `where` condition becomes one line. Rows whose counter cells other
  than the amounts are equal share one counter line carrying the sum of their amounts.
- Amounts are written exactly; a value the line's decimal property cannot hold without rounding
  refuses the run. A run in which no row qualifies is refused. Every refusal is decided before the
  target is written.
- A second run is refused while the first run's target is live. Once that target is reversed, the
  source may be run again.
- `undo: reverse` adds an undo action on the source that books the target's reversal.
- In rows mode each qualifying row becomes one record carrying the source in its `backReference`.
  All records of one run are written as one unit, and a run in which no row qualifies is refused.
- In rows mode a second run is refused while any record back-references the source. `undo: delete`
  deletes exactly those records, as one unit and through the target's own rules; the source may then
  be run again.
- `closingBalance`, `closingNetDebit` and `closingNetCredit` are derived exactly from the closing
  debit and credit, in both modes. With `closingBalance ne 0`, a settled document carries nothing,
  and the net columns put each open one on the side it stands on.

## Edge rules

Refused when the intent is read:

- a header `map` that does not copy the source's key onto the target (items mode);
- a report that is not `kind: balance`, or that declares `correspondence`;
- an unbound `fromDate` or `toDate`; an unknown column; an amount mapped into a property that is not
  a decimal, or a dimension mapped into a decimal property; an ordering operator against `null`;
- `undo: reverse` when the target declares no `reversals:` entry with `backLink`;
- `items.fromReport` together with `items.from`, `items.refuse`, an `event:` trigger, or a source or
  target owned by another model;
- in rows mode: a `backReference` that is not a to-one relation of the target to the source, or that
  is mapped; `fromReport` together with `items`, `prompt`, `link`, `sourceStatus`, an `event:`
  trigger, or a source or target owned by another model;
- `undo: delete` in items mode, and `undo: reverse` in rows mode;
- the report keys on a scheduled generation (`schedules[].generate`).

A create-from without `fromReport`, `items.fromReport` and `undo` generates exactly as before.

## Prior art / workarounds

A hand-written close and a hand-written carry-forward, each with its hand-written undo and its own
guard against a second run. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8534 (items mode) and
https://github.com/eclipse-dirigible/dirigible/pull/8535 (rows mode).

## Not proposed here

Refusing the `undo: delete` once a later period has postings: a condition over another entity's rows.

## Specification text

**Anchor:** Declarative glue > generates - create-from, a new subsection after "Which source rows
become lines - `items.where` / `items.refuse`", ending with its own subsection "One record per report
row".

#### Lines read from a report - `items.fromReport` and `undo`

A create-from's lines MAY be read from a report instead of the source's child rows. A year close
books the ledger's own balances - one closing entry with a line per result account, taking its year
balance off it, and the opposite on the account it closes into - and every one of those numbers is a
row of a `kind: balance` report:

```yaml
generates:
  - name: closeYear
    from: FiscalYear
    to: Journal
    map: { Year: id, date: endDate }         # the source key onto the target: required
    items:
      fromReport: YearBalance                # a kind: balance report
      parameters: { fromDate: startDate, toDate: endDate }
      to: JournalLine
      where:
        - { field: Account.ClosingAccount, op: ne, value: null }
      map: { Account: Account.id, debit: closingCredit, credit: closingDebit }
      counter: { Account: Account.ClosingAccount, debit: closingDebit, credit: closingCredit }
    undo: reverse
```

A column is named by a report dimension (`Account.id`) or by one of the six balance amounts:
`openingDebit`, `openingCredit`, `debit`, `credit`, `closingDebit`, `closingCredit`. Three more
amounts read the closing balance as one number: `closingBalance` (closing debit minus closing
credit), `closingNetDebit` and `closingNetCredit` (that difference on the side it stands on, zero on
the other). `parameters` binds report parameters to properties of the source record, and binds `fromDate` and `toDate`.
`where` takes `{ field, op, value }` conditions over the columns; `value: null` with `eq` or `ne`
asks whether the cell is empty. `counter` declares the counter line: the rows whose counter cells
other than the amounts are equal share one counter line carrying the sum of their amounts.
`undo: reverse` adds an undo action on the source that books the target's
[reversal](#reversals--a-posted-documents-negated-twin). Once the reversal stands, the closed
document is history and the source may be run again - the year is reopened by the booking an
accountant would make by hand. A reversal that should land in the closed year declares no date
default, so the twin keeps the original's date.

> **Normative.**
> `items.fromReport` MUST name a `kind: balance` report that declares no `correspondence`, and MUST
> NOT be combined with `items.from`, `items.refuse`, an `event:` trigger, or a source or target owned
> by another model. The target's `map` MUST copy the source's key onto the target; a file without it
> MUST be reported as an authoring error, as MUST an unbound `fromDate` or `toDate`, an unknown
> column, an amount mapped into a property that is not a decimal, a dimension mapped into a decimal
> property, and an ordering operator against `null`. The report MUST be run with the values the
> source record holds for the bound properties; a record holding no value for one MUST refuse the
> run, and the report MUST NOT be read with its default window instead. Each row satisfying every
> `where` condition MUST become one line. An amount MUST be written as the exact decimal its
> property holds; a value that would need rounding MUST refuse the run. Rows whose counter cells
> other than the amounts are equal MUST share one counter line whose amounts are their sums. A run in
> which no row qualifies MUST be refused. Every refusal MUST be decided before the target is
> written, so that a refused run writes nothing.
>
> A run MUST be refused while a target back-referencing the source is live: not retired by its
> lifecycle stage, and not reversed by a reversal standing in the status the reversal is offered
> from. Two concurrent runs for one source MUST NOT both succeed. `undo: reverse` requires a
> `reversals:` entry on the target that declares `backLink`; a file without one MUST be reported as
> an authoring error. The undo action MUST book that reversal against the live back-referencing
> target, and MUST refuse a target already reversed, a target not standing in the reversal's `when`
> status, and a source with no live target. `undo: delete` in items mode MUST be reported as an
> authoring error. The derived columns `closingBalance`, `closingNetDebit` and `closingNetCredit`
> MUST be computed exactly from the closing debit and credit. A create-from declaring none of
> `fromReport`, `items.fromReport` and `undo` MUST generate byte-identical output.

##### One record per report row - `fromReport` and `undo: delete`

Next year's opening balances are one record per balance-sheet account - and, where the account keeps
a subledger, per counterparty and open document - carrying its closing balance. Without a header, a
`fromReport` on the generation itself makes every qualifying row one record of its own:

```yaml
generates:
  - name: openingBalances
    from: FiscalYear
    to: OpeningBalance
    fromReport: YearBalance
    parameters: { fromDate: startDate, toDate: endDate }
    where:
      - { field: Account.ClosingAccount, op: eq, value: null }
      - { field: closingBalance, op: ne, value: 0 }
    backReference: Year
    map: { Account: Account.id, Partner: Partner.id, Document: Document.id,
           debit: closingNetDebit, credit: closingNetCredit }
    undo: delete
```

`parameters` and `where` read the report as in items mode. `map` reads the report's columns, as
`items.map` does, and `defaults` sets literals; `backReference` names the record's relation to the
source - written on every record, never mapped. `undo: delete` adds an undo action on the source.
A settled document nets to nothing - its closing debit and credit are equal - so
`closingBalance ne 0` leaves it out, and the net columns carry each open one forward on the side it
stands on.

> **Normative.**
> `backReference` MUST name a to-one relation of the target to the source, MUST NOT be mapped, and
> every generated record MUST carry the source in it. The report MUST be run with the source record's
> values as in items mode; each qualifying row MUST become one record, its amounts written as the
> exact decimal their properties hold, or the run refused. The records of one run MUST be written as
> one unit, every refusal decided before the first is written, and a run in which no row qualifies
> MUST be refused. A run MUST be refused while any record of the target back-references the source;
> two concurrent runs for one source MUST NOT both succeed. `undo: delete` MUST delete, as one unit
> and through the target's own rules, exactly the records that back-reference the source, and MUST
> refuse when there is none. `fromReport` MUST NOT be combined with `items`, `prompt`, `link`,
> `sourceStatus`, an `event:` trigger, or a source or target owned by another model, and
> `undo: reverse` in rows mode MUST be reported as an authoring error.

## DSL index

New rows, after `generates.items.refuse`:

| Construct | What it does |
| --- | --- |
| [`generates.items.fromReport`](#lines-read-from-a-report---itemsfromreport-and-undo) | a create-from whose lines are a balance report's result rows, read with parameters bound to the source, filtered by `where`, with an optional counter line; refused on a second run while its target is live |
| [`generates.fromReport` / `generates.backReference`](#one-record-per-report-row---fromreport-and-undo-delete) | one record per balance report row, each carrying the source in its `backReference`; refused on a second run while any of them exists |
| [`generates.undo`](#lines-read-from-a-report---itemsfromreport-and-undo) | an undo action on the source: `reverse` books the target's declared reversal (items mode), `delete` deletes exactly the records one run wrote (rows mode) |
