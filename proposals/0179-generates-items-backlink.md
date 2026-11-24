# A create-from marks each source row it consumed, and never consumes it again

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7956
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8888
- **Builds on:** the create-from's source-row rule (`items.where` / `items.refuse`), the `whenTargetDeleted` rule of a to-one relation

## The problem

Pass-through items are billed by a create-from: the billable expense lines of a project, its
unbilled timesheet days, the materials a job consumed. Each such row must reach the customer's
invoice exactly once.

The mirror form of `items` selects the rows and copies each into a target line, but it never writes
the rows it read. A second run selects the same rows and bills them again. A rule such as
`where: [{ field: billed, op: eq, value: false }]` does not help, because nothing ever sets
`billed`. The source row has no status of its own to flip, so `sourceStatus` does not apply either,
and the header's status says nothing about which of its rows went where: one claim's lines may be
billed to several projects, on several invoices.

## The proposed shape

```yaml
generates:
  - name: invoice-billable-expenses
    from: Project
    to: Invoice
    map: { Project: id }
    items:
      from: ExpenseClaimItem
      to: InvoiceItem
      where:
        - { field: billable, op: eq, value: true }
      backLink: BilledInvoice
      map: { name: description, amount: amount }
```

`ExpenseClaimItem` declares the relation the mark is written into:

```yaml
      - { name: BilledInvoice, kind: manyToOne, to: Invoice, whenTargetDeleted: nullify }
```

## Expected behaviour

- **Each consumed row is marked.** Every source row that becomes a target line has its `backLink`
  relation set to the created target, in the same unit of work as the target and its lines. A
  refusal of anything in that unit takes the marks back with it.
- **A marked row is never selected again.** A row whose `backLink` holds a value is left out of the
  selection, whatever `where` says. It is not counted as unqualified by `refuse` either.
- **Nothing left is a refusal.** A run that finds no unmarked qualifying row is refused, as a run
  whose rule qualifies no row already is. No second document is created.
- **Concurrent runs take turns.** Two runs on the same source cannot both select the same unmarked
  rows.
- **The mark is a system write.** It is made past the row's user-facing immutability, as a
  reversal's back-link on its original is.
- **Deleting the target is the relation's own rule.** `whenTargetDeleted: nullify` clears the marks
  when the generated document is deleted, so its rows are selected by the next run. The default
  `restrict` refuses the delete while any row carries the mark.

## Edge rules

- `backLink` must name a `manyToOne` or `oneToOne` of the items `from` entity whose target is the
  create-from's `to` entity, in the model the create-from's `uses` names. It may not be a
  composition. Anything else is an authoring error.
- `backLink` on items read from a report (`items.fromReport`) is an authoring error: a report row is
  not a stored row a mark could be written on.
- The items need not be the source's own composition. A row of another document that holds a to-one
  to the source, named as the source entity, is selected by that reference.
- A target that is retired (cancelled or voided) rather than deleted keeps the marks. Clearing them
  on retirement is left to a later proposal.
- Without `backLink` an items block behaves exactly as before.

## Prior art / workarounds

A per-row create-from on the line would need the line to have a page and a status of its own, and
would produce one invoice per line. Marking the rows from a separate action after the invoice exists
runs in another transaction, so a failure between the two leaves billed rows unmarked. Reference
implementation: https://github.com/eclipse-dirigible/dirigible/pull/8888.

## Specification text

**Anchor:** generates — create-from > Which source rows become lines — `items.where` /
`items.refuse`, after its normative block.

#### Consumed rows — `items.backLink`

The mirror form of `items` MAY name a to-one relation of the items `from` entity to the target. Each
source row that becomes a line is then marked with the created target, and a marked row does not
become a line again:

```yaml
    items:
      from: ExpenseClaimItem
      to: InvoiceItem
      where:
        - { field: billable, op: eq, value: true }
      backLink: BilledInvoice           # a to-one of ExpenseClaimItem to Invoice
      map: { name: description, amount: amount }
```

What deleting the generated document does to the marks is the relation's `whenTargetDeleted`:
`nullify` frees the rows for the next generation, and `restrict` refuses the delete while a row is
marked.

> **Normative.** A conforming generator MUST set the `backLink` relation of every source row that
> becomes a target line to the created target, in the same unit of work as the target and its
> lines; a refused generation MUST leave every row unmarked. A source row whose `backLink` holds a
> value MUST NOT become a line and MUST NOT be counted as unqualified by `refuse`. A generation that
> finds no unmarked row satisfying `where` MUST be refused as a rule that qualifies no row is.
> Concurrent generations from the same source MUST NOT both select the same unmarked row. The mark
> MUST be written regardless of the row's user-facing immutability. Each of the following MUST be
> reported as an authoring error: a `backLink` that is not a non-composition `manyToOne` or
> `oneToOne` of the items `from` entity to the create-from's `to` entity in the model its `uses`
> names; and `backLink` on items read from a report.

## DSL index

| Construct | What it does |
| --- | --- |
| [`generates[].items.backLink`](#generates--create-from) | a to-one of the items `from` entity to the target: each row that becomes a line is set to the created target in the same unit of work, and a row already set never becomes a line again |
