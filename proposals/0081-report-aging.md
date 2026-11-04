# A report buckets open documents by days past due as of a report date - `aging:`

- **Status:** released in [1.13](../versions/1.13.md)
- **Issue:** https://github.com/IntentFile/intent-specification/issues/142, https://github.com/eclipse-dirigible/dirigible/issues/7888
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7896

## The problem

An aged receivables or payables report - the columns *Not yet due, 0-30, 31-60, 61-90, 90+* per
customer, as of a chosen date - cannot be declared in 1.11. A report's dimensions become the
grouping and its measures the aggregates; nothing puts age buckets in **columns**, measures age from
a date other than today, sums an open amount per bucket, or tells a document due tomorrow apart from
one overdue by a day. 1.11 has no `aging` key.

## The proposed shape

```yaml
reports:
  - name: AgedReceivables
    source: SalesInvoice
    dimensions: [Customer.name]
    aging:
      due: due                  # a temporal field: when the document fell due
      date: issuedOn            # optional: leaves out documents dated after the as-of date
      amount: "total - paid"    # a numeric field, or numeric fields joined by + / -
      buckets: [30, 60, 90]     # ascending, positive; a leading 0 is accepted and ignored
      # asOf: reportDate        # the as-of parameter's name, default asOf
```

## Expected behaviour

After the dimensions and any measures, the report has the columns *Not Yet Due*, *Overdue 0-30*,
*Overdue 31-60*, *Overdue 61-90*, *Overdue 90+* and *Open Amount*, each the sum of the open amount
over the rows the bucket holds:

- The buckets split every row with no gaps and no overlaps, so they always add up to *Open Amount*.
- *Not Yet Due* holds a document due **after** the as-of date, and one with no due date. A document
  due on the as-of date is 0 days past due; one 30 days past due is in *Overdue 0-30*, one 31 days
  past due in *Overdue 31-60*.
- The **as-of date** is a date parameter the report declares on its own behalf, defaulting to today
  on each read.
- Only open documents count: the open amount is not zero, so a settled document is left out and an
  overpaid credit balance stays in. With `date:`, documents dated after the as-of date are left out
  too.
- An aging report aggregates, so its lifecycle `scope` defaults to `live`: a draft or voided invoice
  is not a receivable.

## Edge rules

- Each is an authoring error: a `due` or `date` that is not a temporal field; an `amount` outside the
  `field (+|-) field` grammar, or naming a field that is not numeric; thresholds that do not ascend or
  are not positive; an as-of name that is not a plain, unreserved identifier, or that an authored
  parameter already uses; `aging` on a report that declares a ledger `kind`.
- Limits: a report still has one source, so there is no union of invoices, debit notes and credit
  notes; and the open amount is the document's **current** one - a past as-of date re-buckets today's
  open amounts, it does not replay the payments made up to that date.
- A report without `aging` generates as in 1.11.

## Prior art / workarounds

An age-bucket dimension, which puts the bucket in rows, always measures from today, sums nothing on
its own and counts a document due tomorrow among the first overdue bucket; or a hand-written query.
Reference implementation: https://github.com/eclipse-dirigible/dirigible/pull/7896.

## Specification text

**Anchor:** Presentation > reports > parameters - user-set inputs, a new subsection after it.

#### aging - open documents by days past due

An aged receivables or payables report puts the age of each open document in **columns**: what is
not yet due, and what is overdue by each band of days, as of a date the reader chooses. `aging`
declares those columns over the report's own grouping:

```yaml
reports:
  - name: AgedReceivables
    source: SalesInvoice
    dimensions: [Customer.name]
    aging:
      due: due                  # when the document fell due
      date: issuedOn            # optional: a document dated after the as-of date is not yet open
      amount: "total - paid"    # the open amount: a numeric field, or numeric fields joined by + / -
      buckets: [30, 60, 90]     # the band thresholds in days
      # asOf: reportDate        # the as-of parameter's name, default asOf
```

After the dimensions and any measures, the report gains one column for documents not yet due, one
per band - *Overdue 0-30*, *Overdue 31-60*, *Overdue 61-90* for the thresholds above - one beyond
the last threshold (*Overdue 90+*), and *Open Amount*. Each is the sum of the open amount over the
documents it holds. A document due after the as-of date, or with no due date, is not yet due; one
due on the as-of date is 0 days past due, so it falls into the first band, and a document exactly on
a threshold belongs to the band that threshold closes. `buckets` ascend and are positive; a leading
`0` is accepted and ignored.

The as-of date is a date [parameter](#parameters--user-set-inputs) the report declares on its own
behalf, named `asOf` unless `asOf:` names it otherwise, whose value when the reader leaves it empty
is the current date at each read. Only open documents count: a document whose open amount is zero
is left out, while a negative open amount - an overpaid credit balance - stays in. With `date:`, a
document dated after the as-of date is left out as well. An aging report aggregates, so its
[lifecycle scope](#lifecycle-scope) is that of any aggregating report: without `scope` it counts the
`live` rows.

The open amount is the document's **current** one: a past as-of date re-buckets today's open amounts
and does not replay the payments made up to that date. A report has one source; an aging over
several document kinds is not expressed by this construct.

> **Normative.**
> The bands MUST partition the rows: every row the report counts falls into exactly one of the
> not-yet-due column and the band columns, so that their sum equals *Open Amount* in every row of
> the report. A document due after the as-of date, and one with no due date, MUST be not yet due; a
> document due on the as-of date MUST be 0 days past due. Days past due MUST be counted in calendar
> days between dates. The as-of parameter MUST default to the current date at each read, never to a
> date fixed at generation. A row whose open amount is zero MUST NOT be counted; with `date:`
> declared, a row dated after the as-of date MUST NOT be counted.
> Each of the following MUST be rejected as an authoring error: a `due` or `date` that is not a
> temporal field of the source; an `amount` that is not a field, or fields joined by `+` and `-`, or
> that names a field that is not numeric; `buckets` that do not ascend or hold a threshold that is
> not positive, other than a leading `0`; an as-of name that is not usable as an identifier, that
> the implementation reserves, or that an authored parameter of the report already uses; and `aging`
> on a report that declares a ledger `kind`.

## DSL index

| Construct | What it does |
| --- | --- |
| [`reports.aging`](#aging--open-documents-by-days-past-due) | the aged receivables / payables columns - open amounts not yet due and by bands of days past due, as of a reader-chosen date, summing to the open amount |
