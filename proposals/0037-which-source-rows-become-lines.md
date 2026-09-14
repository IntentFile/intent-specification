# Which source rows become lines

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7091,
  https://github.com/eclipse-dirigible/dirigible/issues/7224 (the refusal is decided before the
  header is saved), https://github.com/eclipse-dirigible/dirigible/issues/7225 (a status name over a
  cross-model source), https://github.com/eclipse-dirigible/dirigible/issues/7251 (the same rule in
  a scheduled query)
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7164,
  https://github.com/eclipse-dirigible/dirigible/pull/7277,
  https://github.com/eclipse-dirigible/dirigible/pull/7300,
  https://github.com/eclipse-dirigible/dirigible/pull/7269

## The problem

A create-from's mirror `items:` block clones EVERY row of the source document into a target line.
There is no way to say which rows qualify - the block carries `from`, `to`, `map` and `defaults`,
and nothing else. So an invoice generated from a project month bills every member timesheet it
holds:

```yaml
generates:
  - name: invoice-from-timesheet
    from: ProjectTimesheet
    to: SalesInvoice
    items:
      from: EmployeeTimesheet
      to: SalesInvoiceItem
      map: { name: employeeName, quantity: totalHours, price: rate }
```

Two failures, both routine:

- A member timesheet still DRAFT, SUBMITTED or REJECTED is billed at the same footing as an APPROVED
  one. Hours nobody has approved reach the customer's invoice, and nothing anywhere says so.
- A member timesheet with no day allocations carries no hours at all, so the line mapped from it is
  missing a value its target requires. The whole generation is refused - correctly, and far better
  than a header-only invoice - but the clerk cannot invoice the month at all until someone deletes
  the empty row by hand. One person who never filed a day blocks the billing of everyone who did.

"Invoice the approved month" is the one flow a billing clerk runs, so the module can either bill
unapproved hours or not bill at all. The shape is not specific to timesheets: every mirror
generation has it - proforma to invoice, quotation to order, order to invoice, delivery note to
invoice.

Row-level validation cannot stand in for it. A `checks:` entry lives on an entity and says whether a
row is legal; this is a question about which legal rows belong in THIS document.

## The proposed shape

A rule on the `items:` block, in the shape a scheduled query already uses:

```yaml
    items:
      from: EmployeeTimesheet
      to: SalesInvoiceItem
      where:
        - { field: Status,     op: eq, value: APPROVED }   # only approved member timesheets
        - { field: totalHours, op: gt, value: 0 }          # an empty one is not a line
      map: { name: employeeName, quantity: totalHours, price: rate }
```

and, where an unqualified row must stop the generation instead of being left out of it, an authored
refusal:

```yaml
    items:
      from: EmployeeTimesheet
      to: SalesInvoiceItem
      where:
        - { field: Status, op: eq, value: APPROVED }
      refuse: "Member timesheet is not approved"
      map: { name: employeeName, quantity: totalHours, price: rate }
```

## Expected behaviour

- Every condition must hold for a source row to become a target line. `op:` is one of `eq`, `ne`,
  `gt`, `ge`, `lt`, `le`, `like` - the operators a scheduled query's `where` takes - and the value
  may be a moment (`CURRENT_DATE`, `CURRENT_TIMESTAMP-PT30M`), resolved against the clock of the run
  that generates, not of the generation of the code.
- **Skipping is the default.** Without `refuse:`, a row that fails the rule is simply not a line.
- **With `refuse:`, an unqualified row refuses the whole generation** with the authored message,
  and the message identifies the rows that failed. Nothing is created: the target's header, its
  lines and any completion hook on the source are one unit, so a refusal leaves the source exactly
  as it was.
- Both refusals - an unqualified row under `refuse:`, and a rule that qualifies no row - are
  decided **before the target header is built**, not rolled back after it. The rule depends on the
  source rows alone, so nothing else needs to exist to decide it; and a header that is saved and then
  undone has side effects a rollback does not reach - a number taken from a continuous series, a
  trail entry for a document that never existed. A refused generation consumes no number and records
  nothing.
- Which of the two an unqualified row deserves is a property of the document. Quietly dropping a
  rejected line from an invoice and quietly billing it are both wrong, for different months, so the
  author says which - and skipping is what a rule with no message means.
- **A rule that qualifies no row at all refuses either way**, naming the source. A document of no
  lines is not the document that was asked for, and it is the harder of the two failures to notice:
  it exists, it counts as the period's billing, and it is empty.
- An items block with no `where:` behaves exactly as it did - every row is a line.

## Edge rules

- `refuse:` requires `where:`. Without conditions no row is ever unqualified, so the message is a
  promise nothing can keep.
- `field:` names a field or a to-one relation of the items `from:` entity. A name it does not
  declare is refused: unlike a scheduled query, whose source row may be owned by another model, the
  rule reads the row being cloned, so an unresolvable name could only ever be a condition rejected
  at the first click.
- A condition naming the source item's own status relation may use the **seeded status name** rather
  than its numeric id, as every other status reference may. It is resolved on the ITEM's own
  nomenclature, not the document header's - resolving against the header's lifecycle would take an
  id out of the wrong nomenclature and filter on it silently. A value that no status can equal - a
  misspelled name, a blank, a moment - is refused by name rather than rendered into a rule that
  matches nothing. The same resolution and the same refusal apply to a scheduled query's `where`,
  the construct this rule is modelled on, so the two cannot drift apart.
- When the items source is **owned by another model**, its nomenclature is seeded there, so a status
  in the rule is referenced by its **numeric seed id**, as every other cross-model status reference
  is; a name is refused, naming the relation and the owner model, rather than left unresolved as a
  rule that matches nothing on every click.
- A moment value must match the compared field's shape: a date field takes `CURRENT_DATE` and a
  date-only offset, a timestamp field `CURRENT_TIMESTAMP`. A moment against a non-temporal field is
  refused rather than compared as text.
- The rule is about the mirror form only. The computed form - a fixed set of synthetic lines whose
  cells are expressions over the source record - has no source rows to select from, and guards its
  lines individually with its own `when` cell.
- An absent value is not a match. A condition of `gt 0` therefore excludes a row whose field is
  empty, which is the "an empty row is not a line" case stated directly.

## Prior art / workarounds

Deleting or filling the offending rows by hand before pressing the button - which is what the
refusal of a required value forces today, and which no one can do for a document generated by a
schedule or an event rather than a click. Adding a status guard to the source document instead
("only generate from an APPROVED project month") states a different rule: it refuses the whole
month because one member's line is not ready, rather than billing the lines that are.
