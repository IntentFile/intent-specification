# A report may declare its own user-set parameters

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/53
- **Reference implementation:** https://github.com/eclipse-dirigible/dirigible/issues/6357

## Why

A report is rarely read once. It is read for a period, for amounts above a threshold, for a
counterparty whose name someone half-remembers — inputs the *reader* chooses at the moment of
reading.

The format has no way to say so. `filter` is fixed when the report is generated, and the only
runtime narrowing a reader gets is whatever per-column filtering the surface happens to offer over
the columns the report already displays. Two consequences follow, and both are routine:

- **A period is not a column.** "This quarter" over an invoice date means two bounds the reader
  sets. Today the model can express either every row or one hard-coded window, so an application
  ends up carrying one report per period, or none.
- **The interesting filter is often on a field the report does not show.** A receivables list
  displays customer, number and balance; the reader wants only the ones whose *internal owner* is
  themselves. A filter over the displayed columns cannot reach that field, so the report grows a
  column nobody wanted in order to be filterable by it.

Only one report kind escapes this, and by hardwiring: the balance report, whose opening/period/
closing shape *requires* a From and a To, declares those two bounds itself. That is the proof the
mechanism belongs to reports in general — a balance report is just the first one that could not be
specified without it.

## What this adds

`parameters` on a report, one entry per input:

```yaml
reports:
  - name: Revenue
    source: Invoice
    dimensions: [date, customer.name]
    measures: ["sum(total)"]
    parameters:
      - { name: fromDate, target: date, op: ge }                 # a From picker
      - { name: toDate, target: date, op: le }                   # a To picker
      - { name: minTotal, target: total, op: ge, initial: "0" }  # an amount threshold
      - { name: customer, target: customer.name, op: like }      # a name search
```

| key | meaning |
| --- | ------- |
| `name` | the input's label source, and the name its value is sent under |
| `target` | the field it filters: a field of the source, or a one-hop `relation.field` path |
| `op` | how the value compares: `ge`, `le`, `eq` or `like` |
| `type` | optional `date` / `timestamp` / `number` / `string` |
| `initial` | the value bound when the reader leaves the input empty |

`target` resolves with the vocabulary a dimension already uses, and a `relation.field` path joins
the related entity exactly as a dimension does — which is what lets a report be filtered by a field
it does not display.

## The normative half

**A parameter is bound on every read.** This is the decision the rest follows from. The alternative
— a predicate that disappears while its input is empty — makes the report's query depend on which
inputs the reader has touched, and a report whose shape changes per read is one no surface can
present, cache or explain.

So the unset case is carried by a **value**, not by the absence of a term, and `initial` is
therefore what the report shows before anyone touches it. Two comparisons have a neutral "any
value" default and need no `initial`: a date `ge`/`le` bound, which widens to all time, and a
`like` search, whose empty pattern matches every value. An `eq` selector and a numeric bound have
none — and defaulting them silently would open the report empty, or narrowed to a value nobody
chose, which is worse than refusing to generate. They must declare `initial`.

**An untouched parameter must not change what the report contains.** That is the promise the neutral
default makes, and it is not free: a plain comparison excludes a row holding *no* value in the
target column, so declaring a from/to window over a nullable date would quietly drop every row
without one — before the reader sets anything. A conforming implementation reads a nullable target
through its empty value so those rows stay in, and the whole-of-time bound still admits them.

**A day is a day.** A `timestamp` target is compared as a date, because the reader's input is a
date. Comparing the stored instant against the start of the chosen day would exclude that day's own
rows from a `le` bound — the classic off-by-a-day that makes a period report quietly wrong at its
right edge.

**`like` matches anywhere in the value.** A reader typing a fragment means "contains"; requiring
pattern syntax in a business input is a leak. It is also what gives the empty default its neutral
meaning.

What is rejected at authoring time: a `target` that is not a field of the source or of the entity
one to-one hop away; a `target` naming a **relation** rather than a field of it (the value would be
an opaque key, and a relation-valued input is a separate construct); a target whose type is not one
a parameter can bind (boolean and long text are excluded); an authored `type` that disagrees with
the target's own type — the declaration is checked, never a conversion; `op: like` against a
non-textual target; a missing `initial` where the comparison has no neutral default; a duplicate
name; and a name that is not usable as an identifier. A balance report may add parameters, but not
redeclare the window it owns.

## Notes

Deliberately **not** part of this proposal:

- **A relation-valued parameter** ("this customer"). The value is a key, so the input is a picker
  over another entity's rows — a different construct, with its own question about which rows are
  offered. `<relation>.<field>` with `op: like` or `op: eq` covers the common case in the meantime.
- **`gt` / `lt` / `ne`.** They have no neutral form: a bound that must exclude a value cannot also
  admit everything, so every such parameter would have to declare a sentinel, which is the thing
  `initial` exists to avoid having to guess.
- **A translated parameter label.** The label is derived from the name, as the balance window's
  already is. Labelling is one question for every generated caption and should be answered once.
- **Parameters on a non-aggregating list.** Nothing here depends on `measures`; the construct is
  specified for reports as a whole, and an implementation that renders a report as a plain list
  binds the same parameters.

## Specification text

The prose below is what a release folds into the next version document, at the anchor given.

**Anchor:** Presentation > reports (after `Lifecycle scope`)

#### parameters — user-set inputs

A report read for a period, a threshold or a name the reader chooses declares those inputs:

```yaml
reports:
  - name: Revenue
    source: Invoice
    dimensions: [date, customer.name]
    measures: ["sum(total)"]
    parameters:
      - { name: fromDate, target: date, op: ge }
      - { name: toDate, target: date, op: le }
      - { name: minTotal, target: total, op: ge, initial: "0" }
      - { name: customer, target: customer.name, op: like }
```

Each entry is one input, offered wherever the report is read, whose value narrows the report.
`target` is the field it filters - a field of the source, or a one-hop `relation.field` path, which
joins the related entity exactly as a dimension does, so a report may be filtered by a field it does
not display. `op` is `ge`, `le`, `eq` or `like`. `type` is optional (`date`, `timestamp`, `number`,
`string`); the target field already types the parameter, so a declared `type` is checked against it.
`initial` is the value used when the reader leaves the input empty, and is therefore what the report
shows before anyone touches it.

> **Normative.**
> A declared parameter MUST narrow the report on every read, using the reader's value when one was
> given and `initial` otherwise; a report MUST NOT vary the shape of its query according to which
> inputs the reader has touched.
> An untouched set of parameters MUST NOT change which rows the report contains, including rows
> holding no value in a target column.
> `initial` MUST be declared unless the comparison has a neutral value that admits every row - a
> date `ge`/`le` bound and a `like` search; `op: eq` and a numeric bound MUST be rejected without it.
> `op: like` MUST match the value anywhere within it, and MUST be rejected against a target that is
> not textual.
> A `timestamp` target MUST be compared by date, so that a `le` bound includes the whole of the day
> given.
> `target` MUST resolve to a field of the source or of the entity one to-one relation hop away; a
> relation itself, and a target whose type a parameter cannot bind, MUST be rejected.
> A declared `type` that disagrees with the target's type MUST be rejected.
> A duplicate parameter name within one report MUST be rejected, as MUST a name that is not usable
> as an identifier or that the implementation reserves.
> A balance report MAY declare further parameters, but MUST NOT redeclare the window bounds it owns.

**Anchor:** Appendix A: DSL index

| [`reports.parameters`](#parameters--user-set-inputs) | user-set inputs a report is read with - a period, a threshold, a name |
