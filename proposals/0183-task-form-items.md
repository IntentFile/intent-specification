# A task form shows the lines of the record it is about - `items:` on a form

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7907
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8892

## The problem

A form's `fields` reach the bound entity's own fields and one-hop `relation.field` paths. A
document's lines are neither: they are the rows of a composition child that reference the
document. So the approval task of a sales invoice shows its number, its customer and its total, and
the approver approves an amount without seeing what is billed. The Issue and Confirm tasks of the
same document have the same gap.

## The proposed shape

```yaml
forms:
  - name: ApproveSalesInvoice
    forEntity: SalesInvoice
    fields: [number, date, Customer, total, Status]
    items: { entity: SalesInvoiceItem, fields: [name, quantity, price, vatRate, total] }
    actions: [approve, reject]
```

## Expected behaviour

- **The lines are a read-only table under the fields**, one column per entry of `items.fields`, in
  order, each value presented as the same field would be presented on the form (a number to its
  scale, a date in the reader's date format). The table has no row actions.
- **The rows are the child's rows that reference the task's record**, read when the form is
  opened - so the lines added after the process started are shown, as the header fields show the
  record's current values.
- **The rows are read with the access of the person who opens the form**, through the same gate
  that governs reading the child's rows anywhere else. The form widens no access: a person who may
  not read the lines is not shown them, exactly as a person who may not read the record is not
  shown its current fields.
- A form declaring no `items:` generates exactly as before.

## Edge rules

- `items` requires `forEntity`.
- `items.entity` MUST be a composition child of `forEntity`, declared in the same model, through a
  `manyToOne` composition relation; any other entity is refused.
- Every `items.fields` entry MUST be a field the child declares itself. A relation, a
  `relation.field` path or an unknown name is refused. At least one field is required.
- `items` is shown only on the form of a user task, and the processes using that form MUST be
  triggered on `forEntity`: the record a task is about is the process's trigger record, so a form
  bound to another entity has no record to list the lines of. Any other use is refused.

## Prior art / workarounds

A hand-written task form, or a `relation.field` per line - which a variable number of lines cannot
be. The read-only lines register a document's own page shows is the presentation this follows.
Reference implementation: https://github.com/eclipse-dirigible/dirigible/pull/8892.

## Specification text

**Anchor:** Processes & forms > forms, after the paragraph and normative block on a cross-model
`relation.field`.

#### items — the lines of the record

A task form may show the **lines** of the record it is about - the rows of a composition child of
`forEntity` that reference the record - as a read-only table under its fields:

```yaml
forms:
  - name: ApproveSalesInvoice
    forEntity: SalesInvoice
    fields: [number, Customer, total]
    items: { entity: SalesInvoiceItem, fields: [name, quantity, price, total] }
    actions: [approve, reject]
```

`items.entity` names the child, `items.fields` its own fields, one column each. The rows are read
when the form is opened, so the approver sees the lines as they stand, including those added after
the process started.

> **Normative.**
> A `forms` entry declaring `items:` MUST be refused unless it declares `forEntity`, `items.entity`
> is a composition child of `forEntity` declared in the same model through a `manyToOne`
> composition relation, `items.fields` lists at least one field and every entry is a field the
> child declares itself, and the form is the form of a user task in processes that are all
> triggered on `forEntity`. A conforming generator MUST render the lines read-only, one column per
> listed field in order, each value presented as the same field is presented on the form. The rows
> MUST be the child's rows referencing the task's record, read when the form is opened, and MUST be
> read with the access of the person opening it, through the same gate that governs reading the
> child's rows elsewhere: a person who may not read them MUST NOT be shown them, and the form MUST
> NOT widen that gate. A form declaring no `items:` MUST generate exactly as before.

## DSL index

| Construct | What it does |
| --- | --- |
| [`forms[].items`](#items--the-lines-of-the-record) | a task form's read-only table of the lines of its record - a composition child of `forEntity` and the child's own fields |
