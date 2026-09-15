# A form field may reach across models

- **Status:** released in [1.8](../versions/1.8.md)
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7093
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7118

## The problem

A form's `fields:` may name a `relation.field` — a field of a one-hop to-one relation, shown
read-only next to the record's own fields (`customer.name`). The current text says the hop is one
hop and to-one, but not which models it may reach. Implementations read that silence as
*local-only*: the relation's target had to be an entity declared in the same file.

The relation a document's form most needs to read a field of is its **counterparty**, and in a
multi-model application the counterparty is almost always owned by another model. A sales-invoice
model reuses the customer from a customers model; its Send task form wants to show the recipient
address, so the clerk sees an empty one *before* pressing Send:

```yaml
forms:
  - { name: SendSalesInvoice, forEntity: SalesInvoice,
      fields: [number, Customer, Customer.email, SentMethod, Status], actions: [send] }
```

A conforming generator refused it:

```
form [SendSalesInvoice] field [Customer.email] references unknown field [email] on [Customer]
```

`Customer` is a cross-model to-one (`{ name: Customer, kind: manyToOne, to: Customer, model: customers }`),
and the *same* path already resolves cross-model elsewhere in the very same file — as a
[`notify`](#notifications) recipient (`to: Customer.email`) and as a `languageFrom` source
(`Language.code`). The form was the one surface that looked the hop up against the local entities
only, so the one relation a billing document's form most needs was the one it could not show.

## The proposed shape

No new key. The `relation.field` entry a form already takes is allowed to cross a model boundary,
provided the relation is declared the way every cross-model relation is — pointing at an alias
listed in `uses:`.

```yaml
uses:
  customers: { model: customers }            # the model that owns Customer

entities:
  - name: SalesInvoice
    fields:
      - { name: number, type: string, number: true }
      - { name: sentMethod, type: string }
    relations:
      - { name: Customer, kind: manyToOne, to: Customer, model: customers, required: true }

forms:
  - name: SendSalesInvoice
    forEntity: SalesInvoice
    fields: [number, Customer, Customer.email, sentMethod]   # Customer.email reaches into the customers model
    actions: [send]
```

## Expected behaviour

A conforming generator MUST resolve a `relation.field` entry whose relation carries a `model:` alias
against the **owner's model** — the model the alias names in `uses:` — at generation time, exactly as
it already resolves a cross-model dropdown, a cross-model `notify` recipient or a cross-model
roll-up parent:

- the field named after the dot MUST exist on the owner's entity; its **type** is the owner's
  declaration of it, and the control is typed from that (an owner `email` string renders as text,
  an owner `decimal` as a number, an owner `date` as a date), not from anything the local file says;
- the generated form renders the value **read-only**, as a resolved value of the related record,
  the same way a local `relation.field` does — the related record is loaded for the form, not typed
  into it;
- the value shown is the owner's **current** record at the moment the form is produced (for a task
  form, when the task is created), read through the owner's own service, so it is not a copy taken
  when the local document was saved.

## Edge rules

- **Declared in `uses:`.** The relation's `model:` MUST name an alias declared in the file's `uses:`
  block. A `model:` that is not declared there is refused when the file is read, with the alias
  named — the rule every cross-model relation already carries.
- **Validated against the owner, at generation.** The hop is checked against the owner's model, not
  against the local document. A field the owner's entity does not declare MUST be **refused** with
  a message naming the form, the path and the owner entity (`form [SendSalesInvoice] field
  [Customer.email] references unknown field [email] on [Customer]`). It MUST NOT be dropped
  silently: a control bound to a value nothing ever provides is indistinguishable from a record
  that has no value, and the reviewer would act on the emptiness.
- **One hop only.** `Customer.email` is allowed; `Customer.Country.name` is not — a second hop is
  refused with a clear message, cross-model or not. A cross-model relation reaches the owner's
  record, which carries the owner entity's own fields but not its relations, so there is nothing to
  walk on from there (the same limit the assignee `path` states for its last segment).
- **To-one only.** The relation must be `manyToOne` or `oneToOne`; every cross-model relation already
  is, since a detail cannot be owned across models.
- **A read-only projection.** A cross-model `relation.field` is a projection of the owner's record.
  It MUST NOT be listed in `editable:` — that key takes a plain field of `forEntity` or one of its
  to-one relations, and a dotted path is neither; the file does not own the field, so the form could
  not write it back. A generator refuses the entry naming the form and the field. Note that the
  bare cross-model relation (`Customer`) is refused in `editable:` as well, by the existing rule
  for task-form pickers: its options live in the owner's model, not in the local flow's context.
  Choosing a different counterparty is done on the document, outside the task form.
- **The same path in a decision.** Where a process decision condition walks the same one hop
  (`Customer.creditLimit > 10000`), a cross-model relation resolves by the same rule: against the
  owner's model, a missing field refused rather than skipped. One resolution serves both.
- **Where the current text disagrees.** The current forms section types controls "by looking each
  field up against the bound entity"; for a cross-model hop the lookup continues into the owner's
  model. The Specification text below states that widening.

## Prior art / workarounds

Before the reference implementation resolved the hop, the module dropped the path from both forms
and made the gap visible with a pre-Send gate instead — a delegate that checks the recipient address
and parks the document in a hold status when it is empty. That is heavier than showing the address
(a step, a status and a task, where a single read-only control would do), and it tells the clerk
*after* Send what the form could have shown *before*.

The same one-hop path is already cross-model for a `notify` recipient (`to: Customer.email`), for a
`languageFrom` source (`Language.code`), for a report dimension (a cross-model relation joins the
owner's table) and for a roll-up parent. The proposal borrows that rule for forms rather than
inventing one; the reference implementation resolves all of them against the owner's model at
generation time.

## Specification text

**Anchor:** Processes & forms > forms — the YAML comment on the `fields:` line, and a new paragraph
with its Normative block appended after "Generates one form per `forms[]` entry …", before
"actions — custom buttons".

The `fields:` line of the `forms` example becomes:

```yaml
    fields: [orderDate, total, customer.name]   # fields or one-hop relation.field; the relation may be cross-model
```

And after the paragraph that begins "Generates one form per `forms[]` entry":

A `relation.field` entry reads a field of a one-hop to-one relation, read-only — the related record
is loaded for the form rather than typed into it, and the control is typed by that record's
declaration of the field. The relation MAY be **cross-model**: a document's form can show a field of
the entity another model owns (an invoice's Send form showing the customer's address), which is the
same one-hop path a [`notify`](#notifications) recipient takes. A cross-model `relation.field` is a
projection of the owner's record — it cannot be listed in `editable:`, which takes a plain field or
a to-one relation of `forEntity` and never a dotted path.

> **Normative.** For a cross-model `relation.field`, the relation's `model:` MUST be declared in
> [`uses`](#reuse-dont-redefine--uses), and the referenced field is validated against the **owner's**
> model at generation time rather than against the local document. A conforming generator MUST
> refuse a field the owner model does not declare, naming the form, the path and the owner entity,
> instead of dropping the control silently: a control bound to a value nothing ever provides is
> indistinguishable from a record that has no value. The path stays **one hop** — a further segment
> is refused, since a cross-model relation reaches the owner's fields but not its relations — and a
> cross-model `relation.field` listed in `editable:` is refused, since the form cannot write a field
> its model does not own.
