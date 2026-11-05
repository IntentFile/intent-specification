# A create-from reads its lines from a report's rows - `items.fromReport`, with `undo: reverse` (items mode)

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8329
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8534

**Scope.** This proposal is **items mode**: a create-from whose header is built from the source
record as before, and whose lines are a report's result rows. **Rows mode** - one target record per
report row with no header, such as next year's opening balances, together with its delete undo - is
not proposed here and follows in a separate proposal.

## The problem

A year close is generated from the ledger's own balances: one closing entry with a line per result
account, taking the account's year balance off it, and the opposite booked on the account it closes
into. Every one of those numbers is a row of a `kind: balance` report over the year's window. No
construct writes records from report rows: a create-from copies one record's header and its child
rows, a reversal copies one entry's lines, a posting books one entry per source event. So the close
can only be written by hand, together with its undo and its guard against a second run.

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
  `closingCredit`.
- `items.where` - optional `{ field, op, value }` conditions over report columns; `op` is
  `eq` / `ne` / `gt` / `ge` / `lt` / `le`; `value: null` with `eq` / `ne` tests for an empty cell.
- `items.counter` - optional: the cells of a counter line, in the shape of `items.map`.
- `items.defaults` - optional literals set on every line and counter line.
- `undo` - optional, `reverse`: an undo action on the source record.

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

## Edge rules

Refused when the intent is read:

- a header `map` that does not copy the source's key onto the target;
- a report that is not `kind: balance`, or that declares `correspondence`;
- an unbound `fromDate` or `toDate`; an unknown column; an amount mapped into a property that is not
  a decimal, or a dimension mapped into a decimal property; an ordering operator against `null`;
- `undo: reverse` when the target declares no `reversals:` entry with `backLink`;
- `items.fromReport` together with `items.from`, `items.refuse`, an `event:` trigger, or a source or
  target owned by another model.

A create-from without `items.fromReport` and `undo` generates exactly as before.

## Prior art / workarounds

A hand-written close, its hand-written undo and its own guard against a second run. Reference
implementation: https://github.com/eclipse-dirigible/dirigible/pull/8534.

## Specification text

**Anchor:** Declarative glue > generates - create-from, a new subsection after "Which source rows
become lines - `items.where` / `items.refuse`".

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
`openingDebit`, `openingCredit`, `debit`, `credit`, `closingDebit`, `closingCredit`. `parameters`
binds report parameters to properties of the source record, and binds `fromDate` and `toDate`.
`where` takes `{ field, op, value }` conditions over the columns; `value: null` with `eq` or `ne`
asks whether the cell is empty. `counter` declares the counter line: the rows whose counter cells
other than the amounts are equal share one counter line carrying the sum of their amounts.
`undo: reverse` adds an undo action on the source that books the target's
[reversal](#reversals--a-posted-documents-negated-twin). Once the reversal stands, the closed
document is history and the source may be run again - the year is reopened by the booking an
accountant would make by hand. A reversal that should land in the closed year declares no date
default, so the twin keeps the original's date.

This is items mode: the report supplies the lines of one target record. One target record per report
row, with no header, is not part of it.

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
> status, and a source with no live target. A create-from declaring neither `items.fromReport` nor
> `undo` MUST generate byte-identical output.

## DSL index

New rows, after `generates.items.refuse`:

| Construct | What it does |
| --- | --- |
| [`generates.items.fromReport`](#lines-read-from-a-report---itemsfromreport-and-undo) | a create-from whose lines are a balance report's result rows, read with parameters bound to the source, filtered by `where`, with an optional counter line; refused on a second run while its target is live |
| [`generates.undo`](#lines-read-from-a-report---itemsfromreport-and-undo) | `reverse`: an undo action on the source that books the target's declared reversal |
