# Reports: a row order, relative-period filters, rows that open their records, and a relation dimension grouped by key

- **Status:** released in [1.13](../versions/1.13.md)
- **Issue:** https://github.com/IntentFile/intent-specification/issues/160, https://github.com/eclipse-dirigible/dirigible/issues/8435, https://github.com/eclipse-dirigible/dirigible/issues/8374, https://github.com/eclipse-dirigible/dirigible/issues/8102
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8457, https://github.com/eclipse-dirigible/dirigible/pull/8385, https://github.com/eclipse-dirigible/dirigible/pull/8476

## The problem

Four gaps in the `reports` grammar of 1.11, each visible on an ordinary dashboard:

1. **No row order.** A report states its dimensions and measures and nothing about the order of its
   rows; only a statement orders its rows by construction. Revenue by month comes back with the
   months shuffled, and a `kind: list` tile's "first `limit` rows" is any five rows rather than a top
   five.
2. **No relative period in `filter`.** 1.11 has relative moments for `schedules[].where` only
   ("`CURRENT_DATE-P7D` is seven days before today"). A report cannot say "this year" or "the last 30
   days", so a year-to-date figure needs a hand-edited date every January.
3. **A row leads nowhere.** A report counts twelve open tickets for an agent, and the reader then
   re-finds those twelve by hand in the ticket list.
4. **A bare to-one dimension groups by the label.** 1.11 says a bare to-one "joins and shows the
   target's label field, not the raw FK id". Grouped by that label, two employees named alike merge
   into one row carrying both figures.

## The proposed shape

```yaml
reports:
  - name: DebitByAccount
    source: JournalLine
    dimensions: [account]                            # grouped by the account's key, shown by its label
    measures: ["sum(debit)"]
    filter: "date >= START_OF_YEAR"                  # also CURRENT_DATE-P30D, START_OF_MONTH,
                                                     #   START_OF_QUARTER, START_OF_YEAR-P1Y
    orderBy: [{ field: "sum(debit)", dir: desc }]    # or the shorthand orderBy: "sum(debit)"

  - name: RevenueByMonth
    source: Invoice
    dimensions: ["month(date)"]                      # no orderBy: ordered by the month, ascending
    measures: ["sum(total)"]

  - name: OpenTicketsByAgent
    source: Ticket
    scope: all
    dimensions: [Assignee]                           # a listed to-one: each row opens its tickets
    measures: ["count(*)"]
    widget: { kind: list, limit: 5, label: Open tickets by agent }
```

## Expected behaviour

**Order.** `orderBy:` names the report's dimensions or measures as authored (`sum(quantity)`,
`product.name`), each with an optional direction. The rows come back in that order, and a filtered
or searched read of the report and a dashboard list tile's `limit` honour it. Without `orderBy`, a
report with a `month()` or `year()` dimension is ordered by it ascending. Ageing buckets are excluded
from that default, because their labels sort as text.

**Relative moments in `filter`.** `CURRENT_DATE`, `START_OF_YEAR`, `START_OF_QUARTER` and
`START_OF_MONTH`, each optionally carrying one signed ISO-8601 date period (`START_OF_YEAR-P1Y`,
`CURRENT_DATE-P30D`), resolve each time the report is read. A moment inside a quoted literal is
text and is left alone.

**Drill-down.** Each row of a report, and of its list tile, opens the source's list filtered by the
row's dimension values, when all of the following hold:

- the report is a plain aggregation: no `kind:`, no ageing buckets, and measures declared;
- no `filter:`, no `parameters:` and no lifecycle scope is in force;
- the source is a same-model entity with its own browse page, not a composition child;
- it has at least one dimension, and every dimension is a column of the source's list that is
  either a bare same-model to-one relation, matched by its label, or an `integer`, `decimal` or
  `boolean` field, matched by its value.

A row with an empty dimension value is plain text. The list, opened with such a filter, applies it
before its first read: a label naming one target matches that target, a label two targets share
matches both, and a label no target carries shows no results with the filter visible. A filter on
another field type, or on a name the list does not know, is ignored.

**Grouping.** A bare to-one dimension groups by the target's **key** and shows its label, so two
targets sharing a name stay two rows. The key is not a column of the report: the rows, search,
count, sums, export and print see the same columns as before, and the count counts the split
groups. The drill-down still matches a relation by its label, so on a label two targets share it
opens both targets' records.

## Edge rules

- `orderBy` refuses a term that is neither a dimension nor a measure of the report, a term listed
  twice and a direction other than `asc` / `desc`, and is
  refused on `kind: statement` and on `kind: ledger` (proposal 0082).
- `orderBy` takes the entity `orderBy` forms of proposal 0086: a list of `{ field, dir }` terms, a
  bare name for an ascending term, and a single term written without the list.
- A `START_OF_` moment the vocabulary does not name, and a time-based period such as `PT1H`, are
  refused when the intent is read.
- Grouping by key applies to every report shape that shows a relation's label - plain, ageing and
  balance reports, a multilingual label, a cross-model target, the correspondence axis. It does not
  apply to plain fields, to `relation.field` paths (a path names exactly what it groups by), to time
  buckets, to listings (which do not group) or to statements.
- Ageing buckets are those of proposal 0081 (`report-aging`).
- A report outside the drill-down conditions generates exactly as before.

## Prior art / workarounds

None within the format: the rows came back in the order the database chose, a period was a
literal date in `filter` edited by hand, a row's records were re-found in the list with its filter
bar, and same-named targets merged into one row. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8457,
https://github.com/eclipse-dirigible/dirigible/pull/8385 and
https://github.com/eclipse-dirigible/dirigible/pull/8476.

## Specification text

**Anchor:** Presentation > reports. The bare to-one bullet of the opening list is amended to:

- a **bare to-one relation** (`customer`) joins and shows the target's label field, not the raw FK
  id - use `customer.id` for the id. An aggregating report groups such a dimension by the target's
  **key**, so two targets sharing a label stay two rows; the key is not a column of the report. A
  cross-model relation joins the owning model's table, so a report can group by an entity another
  module owns;

**Anchor:** Presentation > reports, appended after the paragraph on `filter`.

#### orderBy - the order of a report's rows

`orderBy:` names the report's dimensions or measures as authored, in the forms an entity's
`orderBy` takes:

```yaml
reports:
  - name: DebitByAccount
    source: JournalLine
    dimensions: [account]
    measures: ["sum(debit)"]
    orderBy: [{ field: "sum(debit)", dir: desc }]    # or the shorthand orderBy: "sum(debit)"
  - name: RevenueByMonth
    source: Invoice
    dimensions: ["month(date)"]                      # no orderBy: ordered by the month, ascending
    measures: ["sum(total)"]
```

The order holds on every read of the report, filtered and searched reads included, so a dashboard
[list tile](#dashboard-kpi-widgets) of `limit: 5` shows the first five rows of that order. A report
without `orderBy` that has a `month()` or `year()` dimension is ordered by it ascending; an ageing
bucket has no such default, because its labels sort as text.

> **Normative.**
> Each `orderBy` term MUST name a dimension or a measure of the report exactly as authored; an
> unknown term, a repeated term and a direction other than `asc` or `desc` MUST be reported as
> authoring errors, as MUST `orderBy` on a report of `kind: statement`. The order MUST be applied by
> the query that reads the report, so that a filtered, searched or limited read is a prefix or a
> slice of the one ordered sequence. A report without `orderBy` that declares a `month()` or
> `year()` dimension MUST be ordered by that dimension ascending.

#### Relative moments in a report's filter

A report's `filter` may name a moment relative to the read - the moment vocabulary of a
[schedule's `where`](#relative-moments--a-where-value-offset-from-now), extended by the starts of
the current period:

```yaml
reports:
  - name: DebitThisYear
    source: JournalLine
    dimensions: [account]
    measures: ["sum(debit)"]
    filter: "date >= START_OF_YEAR"          # or START_OF_QUARTER, START_OF_MONTH, CURRENT_DATE
  - name: DebitLastYear
    source: JournalLine
    dimensions: [account]
    measures: ["sum(debit)"]
    filter: "date >= START_OF_YEAR-P1Y AND date < START_OF_YEAR"
```

`CURRENT_DATE`, `START_OF_YEAR`, `START_OF_QUARTER` and `START_OF_MONTH` are dates; each may carry
one signed ISO-8601 date period (`CURRENT_DATE-P30D` is thirty days before today). A moment inside
a quoted literal is text.

> **Normative.**
> A moment in a report's `filter` MUST resolve at each read of the report, never be baked into the
> generated artefact. A `START_OF_` token the vocabulary does not name, and a period that is not a
> date period (`PT1H`), MUST be reported as authoring errors. A moment inside a quoted literal MUST
> be left as written.

#### A row opens the records it counts

A report row links to the source's list, filtered by the row's dimension values, so the twelve
tickets a row counts are one click away. A report carries the link when it is a plain aggregation -
no `kind:`, no ageing buckets, measures declared - with no `filter`, no `parameters` and no
lifecycle scope in force; when its source is an entity of the same model with its own browse page,
not a composition child; and when every dimension is a column of the source's
[list](#control-order) that is a bare same-model to-one relation, matched by its label, or an
`integer`, `decimal` or `boolean` field, matched by its value. A row whose dimension is empty is
plain text.

The list opened this way applies the filter before its first read. A label is matched exactly
against the target's labels: a label one target carries selects it, a label two targets share
selects both - so the link may open more records than a row grouped by key counted - and a label
no target carries shows no results, with the filter in view.

> **Normative.**
> A report meeting every condition above MUST link each row whose dimension values are all present
> to the source's list, filtered by those values, and the list MUST apply that filter before it
> first reads. A relation filter MUST select every target carrying the label, and a label no target
> carries MUST select no rows rather than be ignored. A filter on another field type, and one naming
> a property the list does not know, MUST be ignored. A report failing any condition MUST generate
> exactly as before.

**Anchor:** Presentation > reports > Dashboard KPI widgets, appended to the `kind: list` bullet:

  Each row of the tile opens the source's list as the report's own rows do, under the same
  conditions (see [A row opens the records it counts](#a-row-opens-the-records-it-counts)).

## DSL index

| Construct | What it does |
| --- | --- |
| [`reports[].orderBy`](#orderby---the-order-of-a-reports-rows) | the order of a report's rows, by its dimensions or measures; a `month()` / `year()` dimension orders ascending by default |
| [relative moments in `reports[].filter`](#relative-moments-in-a-reports-filter) | `CURRENT_DATE`, `START_OF_YEAR` / `_QUARTER` / `_MONTH` with an optional date period, resolved at each read |

The `reports` row gains "rows that open the records they count".
