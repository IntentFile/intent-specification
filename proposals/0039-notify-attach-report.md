# A notify block may attach a parameterized report render

- **Status:** draft
- **Issue:** [eclipse-dirigible/dirigible#6931](https://github.com/eclipse-dirigible/dirigible/issues/6931),
  extended by [eclipse-dirigible/dirigible#7030](https://github.com/eclipse-dirigible/dirigible/issues/7030)
- **Implementation:** [eclipse-dirigible/dirigible#6934](https://github.com/eclipse-dirigible/dirigible/pull/6934),
  [eclipse-dirigible/dirigible#7034](https://github.com/eclipse-dirigible/dirigible/pull/7034)
- **Builds on:** [`0021-report-parameters.md`](0021-report-parameters.md) (released in 1.6), whose
  `parameters:` this construct binds; [`0026-naming-a-rendered-document.md`](0026-naming-a-rendered-document.md)
  (released in 1.6), whose `fileName` it reuses; and [`0029-render-language-governs-its-data.md`](0029-render-language-governs-its-data.md),
  which already speaks of "the attached report's rows" and whose language rule this render obeys.

## The problem

A notify block can carry a record's **own** document - `attach: print` renders the invoice and mails
it to its customer, the payslip to its employee, the reminder with the invoice it is about. The rule
is deliberately narrow: `attach` is `print` or `recordPrint`, a generator MUST reject anything else,
and whichever record is rendered must be a document with a line-items child, so a message never
claims an attachment it lacks.

Nothing can carry a **report**. The standard accounts-receivable workflow this blocks is the customer
statement - a period's invoices, credit notes and payments for one customer, with a running balance,
mailed monthly or on demand. Since 1.6 a report declares user-set `parameters:` - a customer, a
from/to period - so the statement is expressible as a report definition, readable and printable on
its page. It cannot leave the system by mail, while its little brother, the per-invoice dunning
reminder, already can. The same gap holds for a supplier's activity list, a monthly usage summary,
an employee's hours for the period: every mailed artifact that is a **slice of rows** rather than one
record's document.

The gap has a second face in a split application. The module that owns the statement report (sales
invoices) reaches `Customer` only through `uses:`, so a schedule over customers there has a
cross-model source - and the current text lets a cross-model schedule source `generate` but is silent
on `notify`, which the reference implementation refused until #7034. The module that owns `Customer`
cannot name a report it does not own. The statement mail had no legal home.

## The proposed shape

A third form of `attach` - a map naming a declared report and binding its parameters from the record
the message is about:

```yaml
reports:
  - name: CustomerStatement
    source: SalesInvoice
    dimensions: [issuedOn, number]
    measures: ["sum(total)"]
    parameters:
      - { name: fromDate, target: issuedOn, op: ge }
      - { name: toDate, target: issuedOn, op: le }
      - { name: customer, target: Customer.name, op: eq, initial: "-" }

schedules:
  - name: monthlyStatements
    cron: "0 0 7 1 * ?"
    entity: Customer
    where: [{ field: openBalance, op: gt, value: 0 }]
    notify:
      to: email
      subject: "Your account statement"
      body: "Dear {name}, please find attached your account statement."
      attach:
        report: CustomerStatement                                    # a declared report of this model
        bind: { customer: name, fromDate: periodStart, toDate: periodEnd }   # parameter <- field of the record
      fileName: "Statement_{name}_{periodStart:yyyyMM}"
      languageFrom: language
```

`report` names a report declared in the same model. `bind` maps a **report parameter** to a field of
the record the message is about, or a one-hop `relation.field` of a to-one relation of it - the same
path vocabulary a `{placeholder}` resolves, against the same record (inside a `forEach`, the row).
The report runs once per message with those values bound, the result is rendered and attached.

The same block is legal at every notify call site - a `notifications[]` entry, a `schedules[].notify`,
a `transitions[].notify`, a sending `serviceTask` - and inside a `forEach` fan-out.

In the split application, the schedule lives in the module that owns the report and names its
cross-model source:

```yaml
uses:
  - { model: customers }
schedules:
  - name: monthlyCustomerStatements
    cron: "0 0 7 1 * ?"
    entity: Customer
    model: customers                          # a cross-model source may notify
    where: [{ field: openBalance, op: gt, value: 0 }]
    notify:
      to: email                               # a field of the source row
      subject: "Your account statement"
      body: "Dear {name}, your statement is attached."
      attach: { report: CustomerStatement, bind: { customer: name } }
```

## Expected behaviour

- **One render per message, scoped by the record.** For each message the block sends, the named
  report is run with every bound parameter set to the value read off the record the message is about,
  and the rendered result is attached. Inside a fan-out that record is the row, so each recipient
  receives the rows that are theirs.
- **The bindings ARE the scoping.** A bound parameter narrows the report exactly as a reader's input
  does on the report page; the report's own `filter:` and `scope:` apply as they do on every read.
  A statement mailed and a statement read on the report page with the same inputs contain the same
  rows.
- **Which parameters must be bound.** Every parameter that declares an `initial` MUST be bound. A
  parameter is bound on every read, so an unbound one rides its `initial` - one fixed slice,
  identical for every recipient: the "whole ledger to one customer" failure, where the mail goes out,
  the attachment is a report, and nothing about it says whose. A parameter **without** an `initial`
  is, by the 1.6 rule, one whose comparison has a neutral any-value default (a date-window bound, a
  `like` search), so leaving it unbound legitimately means "the whole range". A balance report's own
  window bounds are bindable and optional for the same reason.
- **An empty bound value binds nothing.** When the field a parameter is bound to holds no value on
  this record, the parameter behaves as untouched: it takes its `initial` where one is declared and
  admits every row where none is. The render is still produced and the message still sent - a
  customer without a period start receives a statement from the beginning of time, not no statement.
  An author who wants such records skipped filters them out where the block is triggered (`where:` on
  a schedule, `when:` on a notification).
- **The render's language.** `language:` / `languageFrom:` select the render language as they do for
  a document attachment, `languageFrom` resolved against the record the message is about. By
  [proposal 0029](0029-render-language-governs-its-data.md) that language governs the translatable
  values in the attached rows, while what the report **selects** - its `filter:`, its `scope:`, the
  bound comparisons - is evaluated against stored values, so a language changes how the statement
  reads and never which rows it contains.
- **The render's name.** `fileName:` applies as it does for a document attachment, its tokens
  resolved against the record the message is about - a one-hop `relation.field` is admitted, there
  being no anchor-scoped render. Absent a pattern, the attachment is named after the **report and
  the record's identity** (`CustomerStatement Customer 42`), because a mailbox of statements is only
  self-describing when each one names its recipient; a document's own number, which the 1.6 default
  uses, does not exist for a report.
- **The layout is the report's own.** The rows are rendered through a print layout belonging to the
  report, written create-if-absent like a document's print template and hand-owned afterwards - a
  statement sent to a customer is a formatted artifact a later generation must not overwrite. The
  layout is written only for reports something actually attaches, and its header carries the bound
  values, because a table of rows never states which slice it is.
- **Failure semantics follow the call site**, exactly as for `attach: print`: a `transitions[].notify`
  cannot fail its transition, a sending process step fails so the platform's retry applies, a
  schedule row and a fan-out are fail-soft per row. One rule is stricter than for a document: a
  report attachment that cannot be produced **drops the message** rather than degrading it to plain
  text - the bindings are the scoping, and a message without them would carry the wrong rows, which
  is worse than no message.
- **A cross-model schedule source may notify**, and may attach a report of the model the schedule is
  written in. The recipient, the `{placeholder}`s and the `bind:` sources are fields of the source
  row, checked at generation against the owner's model exactly as the `where` fields are; a source
  the owner cannot supply or a mistyped field drops that schedule with a warning, never a job that
  cannot run.

## Edge rules

- **Refused at generation** (an authoring error with a message naming the call site and the key):
  - a scalar `attach` other than `print` / `recordPrint` - the message names all three admitted
    forms, including the map shape;
  - a map `attach` with no `report`, or one naming a report the model does not declare;
  - a key of the map other than `report` and `bind` (the map's vocabulary is closed; `bind`'s own
    keys are the report's parameters);
  - a `bind` key that is not a parameter of that report - reported with a close-match suggestion
    where one exists, never passed on as a filter the report would ignore and mail unfiltered;
  - a `bind` entry with no source;
  - a `bind` source that is not a field of the record the message is about or a one-hop
    `relation.field` of a to-one relation of it - a multi-hop path, an unknown relation, a
    to-many relation, an unknown field of the related entity;
  - a `bind` source in the `record.` scope inside a fan-out - the rows are the recipients and the
    report is scoped by the row; a report about the anchor would be about something other than what
    the message is about. There is no `recordReport` mirror of `recordPrint`: a report attachment is
    always scoped by the record the message is about, because that is what makes the rows the
    recipient's;
  - a report parameter with an `initial` left unbound - the message names the parameter and the
    `initial` every recipient would otherwise be mailed;
  - a report that declares **no** parameters (and is not a balance report, which owns a window) -
    with nothing to bind it renders the same result for every recipient, so the author is told to
    declare the parameters that scope it or to attach it to a schedule that runs once;
  - `language:` together with `languageFrom:`.
- **`bind` may be omitted** when the report declares no parameter with an `initial` - every parameter
  is then a neutral bound and the whole range is legitimately what is mailed. The report must still
  declare at least one parameter.
- **The document requirement does not apply.** The record the message is about need not be a
  document; a report attachment is read off any entity, since the report - not the record - is what
  is rendered. This is the sentence of the current text the construct changes: "`attach` is `print`
  ... or `recordPrint`" and "a generator MUST reject `attach: print` on any other entity" stay true
  of the two scalar forms and say nothing about the map form.
- **`fileName` on a report attachment** resolves against the record the message is about, not "the
  record that block renders" - a report attachment renders no record. `{Version}` is refused, as on
  every notify block. The current fileName text's default ("a document's own number is used where it
  declares one") does not reach a report; the default is the report's name and the record's identity.
- **Inside a fan-out**, `bind:` follows every other bare path and resolves against the row; the row
  need not be a document. The fan-out's fail-soft-per-row rule applies unchanged.
- **A cross-model schedule source** is narrowed to what the owner model can supply; four things are
  refused at generation:
  - a `relation.field` hop off the source row - a foreign entity's relations are known only to its
    owner, the same rule a cross-model `generate map` states;
  - `{recordUrl}` - it composes a route of *this* application while the record belongs to the owner's;
    `{appUrl}` plus the owner's path is the honest form;
  - `attach: print` / `attach: recordPrint` - a document's render is produced in the model that owns
    the document; a report of *this* model is what the lift is for;
  - any construct that writes back to the source row from the notify block, for the same reason.
  The 1.6 schedules text already says a cross-model source's "notify paths are validated against the
  owner's model at generation time" but never says a cross-model source may notify at all, and the
  reference implementation refused it before #7034; the Specification text makes the permission and
  its narrowing explicit.
- **Where the current text contradicts the shipped behaviour.** The Normative block of "The notify
  block — and `attach: print`" says `attach` is `print` or `recordPrint` and, read with the
  `recordPrint` block, that nothing else is admitted; the reference implementation admits the map
  form and its rejection message names it. The fileName text says a notify block's pattern resolves
  "against the record that block renders" and defaults to "a document's own number"; for a report
  attachment it resolves against the record the message is about and defaults to the report name plus
  the record's identity. The Specification text below replaces both sentences.

## Prior art / workarounds

Without the construct, a statement leaves the system by hand: someone opens the report page, sets
the customer and the period, prints, and mails the file - once per customer, once per month. Or a
hand-written scheduled job runs the report's query, formats the rows and calls the mail transport -
a second copy of the report's definition that drifts from the first the day a column is added. Or the
statement is remodelled as a **document** - a `CustomerStatement` entity with generated line items
snapshotting the period's rows - purely so that `attach: print` can carry it; a materialised copy of
what a report already computes, kept in sync by yet another glue entry.

The reference implementation shipped the map form in #6934 and lifted the cross-model schedule
restriction in #7034; the customer-statement mail of its billing suite is the driver of both.

## Specification text

**Anchor:** Declarative glue > The notify block — and `attach: print`, sending the document itself.
The first Normative blockquote's opening sentence ("`attach` is `print` - the record the block is
about - or, inside a fan-out, `recordPrint`.") is replaced by the paragraph and blockquote below; the
rest of that blockquote (the document requirement, `language`, the same-path rule) is unchanged and
now reads as the rule for the two scalar forms.

`attach` takes one of three forms: `print` - the record the block is about; inside a fan-out,
[`recordPrint`](#one-document-many-recipients-attach-recordprint) - the anchor record; or a map
`{ report: <name>, bind: { <parameter>: <path> } }` - a declared **report**, run for this message
with its [`parameters`](#parameters--user-set-inputs) bound from the record the message is about.
The two scalar forms carry a record's own document and require a document. The map form carries a
**slice of rows** - a customer's statement for the period, a supplier's activity list - and requires a
report with parameters: `report` names a report declared in this model, and each `bind` entry sets one
of its parameters from a field of the record the message is about, or a one-hop `relation.field` of a
to-one relation of it - the same paths a `{placeholder}` resolves, against the same record (the row,
inside a `forEach`).

```yaml
    notify:
      to: email
      subject: "Your account statement"
      body: "Dear {name}, please find your account statement attached."
      attach:
        report: CustomerStatement                                     # a report of this model
        bind: { customer: name, fromDate: periodStart, toDate: periodEnd }
      languageFrom: language                                          # the render's language
      fileName: "Statement_{name}_{periodStart:yyyyMM}"               # the render's name
```

> **Normative.** A scalar `attach` other than `print` or `recordPrint`, a map `attach` naming no
> report or an undeclared one, and a map key other than `report` and `bind` MUST be rejected. The
> report MUST be run once per message with every bound parameter set to the value read off the record
> the message is about, and the rendered result attached; inside a fan-out the record is the ROW and a
> `bind` source in the `record.` scope MUST be rejected - a report attachment is always scoped by the
> record the message is about, and the format has no anchor-scoped report form. The record need not be
> a document. Each `bind` key MUST be a parameter of the named report (a balance report's own window
> bounds count as parameters); an unknown key MUST be rejected, not passed on as a filter the report
> ignores. Each `bind` source MUST resolve to a field of the record or to a one-hop `relation.field`
> of a to-one relation of it; a blank source, a longer path, or a path through a to-many relation MUST
> be rejected. Every parameter of the report that declares an `initial` MUST be bound - left unbound
> it would ride its `initial` and mail one fixed slice to every recipient - and a report attachment
> whose report declares no parameters MUST be rejected, since it would render the same result for
> every recipient; a parameter without an `initial` MAY be left unbound and then admits every row.
> A bound value that resolves empty binds nothing: the parameter takes its `initial` where declared
> and admits every row where not, and the message is still sent. The report's own `filter:` and
> `scope:` apply as on every read, so a statement mailed and one read on the report page with the
> same inputs MUST contain the same rows. A message whose report attachment cannot be produced MUST
> be dropped, not sent as text without it: the bindings are what make the rows the recipient's.
> `language` / `languageFrom` select the render's language as for a document attachment, `languageFrom`
> read off the record the message is about, and MUST NOT both be declared; the render's language
> governs the translatable values of the attached rows and never their selection (see
> [multilingual data](#multilingual-data)). The failure semantics of the call site apply unchanged.

The render is laid out by a print layout of the **report's** own, written create-if-absent, only for
reports some block attaches, and hand-owned afterwards - a statement sent to a counterparty is a
formatted artifact, and a later generation MUST NOT overwrite a designed one. The layout's header
carries the bound values, since a table of rows never states which slice it is.

**Anchor:** Presentation > Printable documents > Naming the rendered file — `fileName`. The sentence
"A snapshot's pattern resolves against its **document master**; a notify block's against the record
that block renders." is replaced, and one Normative sentence is appended.

A snapshot's pattern resolves against its **document master**; a notify block's against the record
that block renders - for a report attachment, which renders no record, against the record the message
is about, a one-hop `relation.field` included.

> **Normative.** Absent a pattern, a report attachment MUST be named after the report and the identity
> of the record the message is about - a mailbox of statements is self-describing only when each one
> names its recipient; a document's own number does not exist for a report.

**Anchor:** Declarative glue > schedules. Appended to the paragraph beginning "The schedule's `entity`
may be owned by another model".

A cross-model source may `notify` as well as `generate`, and the block may attach a report **this**
model declares - the module that owns a statement report seldom owns the customer it is about, and
the schedule then has no other home. What the owner alone can supply is out of reach: the recipient,
the `{placeholder}`s, `languageFrom`, `fileName` and the `bind` sources are direct fields of the
source row.

> **Normative.** On a `schedules[].notify` whose source is cross-model a conforming generator MUST
> reject a `relation.field` hop off the source row, `{recordUrl}`, and `attach: print` /
> `attach: recordPrint` - the row's relations, its route and its document all belong to the owner
> model. The recipient, placeholder and `bind` fields MUST be checked against the owner's model at
> generation, and a source or field the owner does not supply MUST drop that schedule with a warning
> naming the owner, never emit a job that cannot run.

## DSL index

| Construct | What it does |
| --- | --- |
| [`attach: { report, bind }`](#the-notify-block--and-attach-print-sending-the-document-itself) | attach a declared report, run per message with its `parameters` bound from the record the message is about - the customer statement, the supplier activity list; every parameter with an `initial` must be bound |
