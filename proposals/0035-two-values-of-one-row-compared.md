# Two values of one row, compared

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7095

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

The only way to express it today is a hand-written calculated action per document type, and an
action **corrects** rather than refuses: it recomputes the due date from the new date and saves. The
record ends up consistent and the person who typed the date is never told it was overruled - which
is a different rule from the one the author wanted, and a class per comparison to state it.

## The proposed shape

A row-level kind that names two of the record's own fields and the relation they must stand in:

```yaml
- name: SalesInvoice
  checks:
    - { kind: compare, field: due,  op: ge, than: date,  message: "Due cannot be before the invoice date" }
    - { kind: compare, field: paid, op: le, than: total, message: "Paid cannot exceed the total" }
  fields:
    - { name: date,  type: date, required: true }
    - { name: due,   type: date }
    - { name: total, type: decimal }
    - { name: paid,  type: decimal }
```

`op:` is one of `ge`, `gt`, `le`, `lt`, `eq`, `ne`, read as `field <op> than`.

## Expected behaviour

- The check is **row-level**, like `exactlyOne`: it holds on every user write and is refused with the
  authored message (400 over HTTP), on every generated surface the record can be written through.
- It therefore takes **no `status` gate**. A rule about two values of one row holds from the first
  save; the gate exists for document-level checks, which would otherwise forbid drafting a document
  line by line.
- `op:` is required. An omitted operator has no defensible default - "not before" and "strictly
  after" are different rules and the wrong guess is silent.
- An **absent operand is not a violation**. A comparison is about two values that exist; whether a
  field may be empty at all is `required:`, which is its own declaration. So a record carrying no
  `due` passes, and starts failing the moment one is entered behind the date.

## Edge rules

- Both operands are the record's own **fields**. A relation is not comparable (a comparison of two
  foreign keys means nothing), and a field of a related record is not in scope for v1.
- The two fields must be in ONE comparison family: both dates, both timestamps, or both numbers of
  any width. A date against a timestamp is refused rather than coerced - what a coercion would have
  to invent (which day boundary, which zone) is exactly what the author has not said.
- Only dates, timestamps and numbers compare. A string, a boolean or a month/week label is refused
  rather than silently ordered lexicographically.
- Numbers compare by **value**, so a `decimal` against a `long`, or two decimals of different scale,
  compare exactly.
- A field compared with itself is refused: the outcome cannot depend on the record.
- Several `compare` entries on one entity are independent and all hold.

## Prior art / workarounds

`calculatedActionOnCreate` / `calculatedActionOnUpdate` with a hand-written class, as described
above: it corrects instead of refusing, costs one class per comparison, and puts a rule that is
plainly declarative into code. A `transitions:` guard can gate a status hop on a comparison, but the
document is already saved by then - the wrong values are in the database and every list, report and
reminder reads them.
