# A count tile over an aggregating report

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7102

## The problem

A report `widget:` of `kind: count` is specified as "the number of records the report yields". A
report that declares `measures:` yields one row per group, so there are two different numbers the
tile could show and the specification names only the one nobody can compute from the result set:

```yaml
reports:
  - name: RequestsByStatus
    source: VacationRequest
    dimensions: [Status]
    measures: ["count(*)", "sum(days)"]
    widget: { kind: count, label: Vacation Requests }
```

The report's rows are `INSUFFICIENT 2`, `DRAFT 7`, `REJECTED 1`, `APPROVED 4` - fourteen requests in
four statuses. A generator that counts the rows of the result set puts **4** on the tile. The number
is wrong, and it is wrong in the direction that hides work: a tile reading 4 next to a list of 14
records looks like a filter, not a defect. On a report grouped by a single dimension value the same
tile reads **1**.

Nothing in the model distinguishes the two readings, so a conforming generator can be correct by the
letter of the specification and still show the number of groups.

## The proposed shape

No new key. The count of an aggregating report is its `count(*)` measure, summed over the rows:

```yaml
reports:
  # aggregating: the tile shows 14 - the count(*) measure summed over the four rows
  - name: RequestsByStatus
    source: VacationRequest
    dimensions: [Status]
    measures: ["count(*)", "sum(days)"]
    widget: { kind: count, label: Vacation Requests }

  # not aggregating: one row IS one record, so the row count is the record count
  - name: OpenRequests
    source: VacationRequest
    dimensions: [number, employee.name, fromDate, days]
    filter: "Status == DRAFT"
    widget: { kind: count, label: Open Requests }
```

## Expected behaviour

- A `kind: count` widget over a report that declares **no** measures shows the number of rows the
  report yields, which is the number of records it reads. Unchanged.
- A `kind: count` widget over a report that declares measures shows its `count(*)` measure **summed
  over the report's rows**, under the same filter, lifecycle scope, parameters and widget pins that
  the report itself applies.
- An aggregating report carrying a `kind: count` widget and **no** `count(*)` measure is rejected at
  generation, naming the report and the fix (declare `count(*)`, or use `kind: value` for a measure
  the report already has). A tile whose only available number is the number of groups must not be
  generated.

## Edge rules

- `count(*)` and `count()` are the same measure; a `count(<field>)` is not it - it counts the rows
  where that field has a value, which is a different number and one an author may want beside the
  total. Only the star form satisfies the requirement.
- A report whose rows are its unit of account by construction - a ledger balance (one row per
  account) or a statement (one row per declared line) - counts its rows and declares no measure, so
  the requirement does not apply to it.
- Widget pins (`at:`) narrow the rows before the sum, so a pinned count is the record count of that
  slice.
- A report that declares measures and no `widget:` is unaffected, as is one whose widget is of kind
  `value` or `list`.

## Prior art / workarounds

The only workaround available today is `kind: value` over the report's own `count(*)` measure - but
`value` reads ONE cell (the first row, or the row the pins select), so on a status-dimensioned report
it shows the count of a single status. There is no authored shape that reaches the total, which is
why this is a defect in the specified behaviour of `count` rather than a request for a new key.
