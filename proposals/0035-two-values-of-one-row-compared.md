# Two values of one row, compared

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7095,
  https://github.com/eclipse-dirigible/dirigible/issues/7338 (the literal operand and the gate)
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7121 (two fields of one
  row), https://github.com/eclipse-dirigible/dirigible/pull/7355 (a literal right-hand side and the
  optional `status:` gate)

## The problem

`checks:` can say that exactly one of several fields is filled (`exactlyOne`), that a document's
line sums balance (`itemsSumEqual`) and that it has lines at all (`itemsMin`). No kind relates two
values of the SAME record, so the most ordinary rule a business document has cannot be declared:

```yaml
- name: SalesInvoice
  fields:
    - { name: date, type: date, required: true }
    - { name: due,  type: date }
```

Nothing says `due` is never before `date`. An invoice dated the 9th with terms of two weeks got a
due date of the 23rd; the date was later moved three months forward and the due date stayed where it
was - saved, issued, and overdue the moment it existed. The same shape recurs across a suite: a
proforma's `due` against its `date`, a validity `to` before its `from`, a delivery date before the
order date, a paid amount above the total.

Nor does any kind relate a value to a **constant**, which is the commonest validation a business
model has: a quantity that must be positive, a percentage that cannot exceed 100, a date that may not
lie in the past. A leave request whose `days` is negative silently deflates the entitlement it is
rolled up into, because the roll-up sums the column as it finds it.

The only way to express either today is a hand-written calculated action per document type, and an
action **corrects** rather than refuses: it recomputes the due date from the new date and saves. The
record ends up consistent and the person who typed the date is never told it was overruled - which
is a different rule from the one the author wanted, and a class per comparison to state it.

## The proposed shape

A row-level kind that names a field of the record and the relation it must stand in to a second
value - either another of the record's own fields (`than:`) or a literal (`value:`):

```yaml
- name: SalesInvoice
  checks:
    - { kind: compare, field: due,  op: ge, than: date,  message: "Due cannot be before the invoice date" }
    - { kind: compare, field: paid, op: le, than: total, message: "Paid cannot exceed the total" }
    - { kind: compare, field: discountPercent, op: le, value: 100, message: "A discount cannot exceed 100%" }
  fields:
    - { name: date,  type: date, required: true }
    - { name: due,   type: date }
    - { name: total, type: decimal }
    - { name: paid,  type: decimal }
    - { name: discountPercent, type: decimal }

- name: VacationRequest
  checks:
    # ungated: holds from the first save
    - { kind: compare, field: from, op: ge, value: "CURRENT_DATE", message: "Leave cannot start in the past" }
    # gated: a zero-day draft may be filled in; submitting one is refused
    - { kind: compare, field: days, op: gt, value: 0, status: SUBMITTED,
        message: "A request must cover at least one working day" }
```

`op:` is one of `ge`, `gt`, `le`, `lt`, `eq`, `ne`, read as `field <op> than` or `field <op> value`.

## Expected behaviour

- The check is **row-level by default**, like `exactlyOne`: it holds on every user write and is
  refused with the authored message (400 over HTTP), on every generated surface the record can be
  written through.
- It takes an **optional `status:` gate**, and the gate's presence decides WHEN the rule is
  evaluated - the same routing `requiredWhen` has. Without one the rule holds from the first save.
  With one it is evaluated when the record is persisted carrying that status - the transition into
  it, not the drafting before it - so "a submitted request covers at least one day" can be stated
  without refusing the draft still being filled in. A gated comparison is refused with the authored
  message to whoever performed the transition, and needs an entity-status relation to read the gate
  from.
- `op:` is required. An omitted operator has no defensible default - "not before" and "strictly
  after" are different rules and the wrong guess is silent.
- Exactly one of `than:` and `value:` is given: a comparison has one right-hand side.
- An **absent operand is not a violation**. A comparison is about two values that exist; whether a
  field may be empty at all is `required:`, which is its own declaration. So a record carrying no
  `due` passes, and starts failing the moment one is entered behind the date. The same holds against
  a literal: a record carrying no `days` is not refused by `days > 0`.

## Edge rules

- The left operand is the record's own **field**; the right one is another of its own fields or a
  literal. A relation is not comparable (a comparison of two foreign keys means nothing), and a
  field of a related record is not in scope for v1.
- Two fields must be in ONE comparison family: both dates, both timestamps, or both numbers of any
  width. A date against a timestamp is refused rather than coerced - what a coercion would have to
  invent (which day boundary, which zone) is exactly what the author has not said.
- A **literal is typed by the field it is compared with**. A numeric field takes a number. A `date`
  or `timestamp` field takes either a **moment** - `CURRENT_DATE`, `CURRENT_TIMESTAMP` or `NOW`,
  with at most one signed ISO-8601 offset (`CURRENT_DATE+P7D`), the vocabulary a schedule's `where`
  already carries, resolved against the clock of the WRITE - or a quoted ISO-8601 date / instant.
  The literal must match the field's shape: `CURRENT_TIMESTAMP` or a time-based offset against a
  `date` field is refused, as a date field against a timestamp field is. A literal that is not a
  value of the field's type (a word against a number) is refused.
- A temporal literal is **quoted**. An unquoted `2026-01-01` is a date object to the YAML loader
  before the intent ever sees it, and is refused with a message saying so.
- Only dates, timestamps and numbers compare. A string, a boolean or a month/week label is refused
  rather than silently ordered lexicographically.
- Numbers compare by **value**, so a `decimal` against a `long`, or two decimals of different scale,
  compare exactly - against a field or a literal alike.
- A field compared with itself is refused: the outcome cannot depend on the record.
- Several `compare` entries on one entity are independent and all hold.

## Prior art / workarounds

`calculatedActionOnCreate` / `calculatedActionOnUpdate` with a hand-written class, as described
above: it corrects instead of refusing, costs one class per comparison, and puts a rule that is
plainly declarative into code. A `transitions:` guard can gate a status hop on a comparison, but the
document is already saved by then - the wrong values are in the database and every list, report and
reminder reads them.

For a comparison against a constant the field had two further workarounds, both worse than the gap:
a hand-edit of the generated validation, which the next regeneration drops silently, and a
create-time calculation that throws - a calculation rather than a refusal, firing only on the field
that declares it and reaching the caller as whatever its exception happens to carry. The gated form
closes a third: a "days > 0 before SUBMITTED" rule mis-authored as an `itemsMin` over a child the
approval step had not created yet, which refused every submission.
