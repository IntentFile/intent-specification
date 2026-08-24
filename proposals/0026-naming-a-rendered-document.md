# Naming a rendered document: `fileName`

- **Status:** released in [1.6](../versions/1.6.md)
- **Issue:**
- **Reference implementation:** https://github.com/eclipse-dirigible/dirigible/issues/6899

## Why

A document is rendered to a file in two places, and the intent can say nothing about what that file
is called. Both names are fixed by the implementation, and in practice they do not even agree with
each other: the versioned copy a document mints on issue tends to be named after the record's
numeric primary key, while the copy a `notify` block attaches is named after the document number the
recipient actually recognises. One document therefore reaches the archive and the counterparty under
two different names.

The name is not cosmetic. It is the only thing a file carries once it leaves the application:

- an **archive** is browsed and searched by file name, and a folder of `Order 42 v1.pdf`,
  `Order 43 v1.pdf` is unusable at the exact moment it matters - a regulator asking for the
  invoices issued to one customer in one month;
- an **attachment** arrives in a mailbox with no application around it, so the name is the whole
  context the recipient has;
- a **hand-off** to an accountant, an auditor or a factoring service is usually a folder of files
  and a naming convention agreed in advance - a convention the model currently cannot express, so
  it ends up being applied by hand, or not at all.

Every one of those wants the same thing: a self-describing name composed of the record's own data -
`SI0000042_20260822_MyCompany_AcmeLtd.pdf`.

## What this adds

`fileName` - a **pattern** of literal text and `{token}` interpolations - on the two declarations
that produce a rendered file: the `function: Snapshot` child, and a `notify` block that attaches a
render.

```yaml
entities:
  - name: SalesInvoiceCopy
    function: Snapshot
    fileName: "{number}_{date:yyyyMMdd}_{company.shortName|company.name}"
    relations:
      - { name: salesInvoice, kind: manyToOne, to: SalesInvoice, composition: true, required: true }

transitions:
  - name: SendInvoice
    forEntity: SalesInvoice
    from: [DRAFT]
    setStatus: SENT
    notify:
      to: customer.email
      subject: "Invoice {number}"
      body: "Please find the invoice attached."
      attach: print
      fileName: "{number}_{customer.name}"
```

The token vocabulary is the one an author already knows - the same paths a notify `subject` or `body`
interpolates, resolved against the record being rendered:

- **`{field}`** and one-hop **`{relation.field}`**, named exactly as the model declares them;
- **`{field:pattern}`** - a date or timestamp field rendered through a date-format pattern;
- **`{A|B}`** - alternative operands, the first non-blank one wins;
- **`{Version}`** - the copy's version, on a snapshot only.

Nothing else: no literals inside tokens, no arithmetic, no conditionals. A file name is a label, and
a label that needs an expression language is a report.

`{A|B}` is there because the optional twin field is the normal shape of the data a name is built
from. A partner carries a short trading name beside its registered name, filled for some partners
and not for others; without a fallback the pattern has to pick one, and every record that happens to
have the other filled produces a worse name than the default did.

## The normative half

**Where the paths resolve.** A snapshot's pattern resolves against its **document master** - the
record whose copy is being minted; the copy row itself holds only the stored file's coordinates.
A notify block's pattern resolves against whatever that block renders: the record it is about, or,
inside a fan-out that attaches the anchor's document, the anchor.

**Sanitization.** An interpolated **value** is sanitized before it enters the name: trimmed, internal
whitespace collapsed to a single separator, and characters a name may not carry removed. The literal
text between tokens is the author's and is emitted verbatim - which is what makes the convention
legible: the author owns the separators, the data owns the parts.

Non-ASCII characters are **kept**. A document in a local language legitimately carries a non-Latin
name, and whether names should be transliterated is an application's data convention, not something
the format may decide by mangling the value.

**Versions.** Two versions of one copy must never share a name. A snapshot pattern that places
`{Version}` itself decides where the version goes; one that does not gets the version appended.

**Blank and unresolvable are different things.** A token whose value is blank renders empty - that is
what `{A|B}` is for, and a name with a missing part is still a name. A token that names a field or a
relation the entity does not have is an **authoring error**, reported before anything is generated.
The two must not be conflated: a silently-dropped token yields exactly the indistinguishable names
this construct exists to remove, and it does so invisibly.

What else is rejected at authoring time: a pattern with no interpolation at all (it would name every
file alike - the failure being fixed); unbalanced or nested braces; a date format on a field that is
not a date or timestamp; a format string the implementation's date formatter does not accept; a
multi-hop path; `{Version}` where no version exists; and a relation hop where the render happens
outside the scope the related record is available in.

**Absent a pattern**, an implementation supplies a default. The default MUST be the same expression
for both renders - a rendered document is one document, and two defaults that disagree are the
defect this proposal starts from. The document's own number is the obvious choice where the document
declares one.

Appendix A gains its rows.

## Notes

Deliberately **not** part of this proposal:

- **A file extension in the pattern.** The extension follows from the render format, which the
  intent does not choose.
- **Directory structure.** Where a file is stored is a storage concern; this names the file.
- **Multi-hop paths.** They would need a second load per render and, more importantly, the composed
  value belongs in a field of the record - the same rule the notify placeholders already follow.
- **Renaming already-stored files.** A pattern change applies to what is rendered afterwards. A copy
  that has been issued is an artefact with a name, and changing it retroactively is the opposite of
  what an immutable copy is for.

## Specification text

The prose below is what a release folds into the next version document, at the anchors given.

**Anchor:** Printable documents (a new subsection, after "Row filtering")

#### Naming the rendered file - `fileName`

A document is rendered to a file in two places - the versioned copy a
[`function: Snapshot`](#attachments-and-snapshots) child mints, and the copy a
[notify block](#the-notify-block--and-attach-print-sending-the-document-itself) attaches. Both accept
a `fileName` pattern: literal text and `{token}` interpolations over the record being rendered.

```yaml
    fileName: "{number}_{date:yyyyMMdd}_{customer.shortName|customer.name}"
```

| Token | Renders |
| --- | --- |
| `{field}` | a field of the rendered record |
| `{relation.field}` | a field of a one-hop to-one relation of it |
| `{field:pattern}` | a date or timestamp field in the given date-format pattern |
| `{A\|B}` | the first non-blank of the listed operands, left to right |
| `{Version}` | the copy's version (a snapshot only) |

Paths use the authored field and relation names - the same vocabulary a notify `subject` interpolates.
A snapshot's pattern resolves against its **document master**; a notify block's against the record
that block renders.

> **Normative.**
> An interpolated value MUST be sanitized before it enters the name: trimmed, internal whitespace
> collapsed to a single separator, and characters the storage or transport cannot carry removed.
> Non-ASCII characters MUST be preserved - whether a name is transliterated is an application's data
> convention, not the format's.
> Literal text between tokens MUST be emitted verbatim: the author owns the separators.
> A token whose value is blank renders empty. A token naming a field or relation the entity does not
> have MUST be rejected at authoring time, not rendered empty - a silently dropped token produces
> exactly the indistinguishable names this construct removes.
> A pattern that interpolates nothing, has unbalanced or nested braces, applies a date format to a
> field that is not a date or timestamp, carries a format the implementation's formatter rejects, or
> names a multi-hop path MUST be rejected.
> `{Version}` MUST be rejected where no version exists. A snapshot pattern that does not place it
> MUST have the version appended: two versions of one copy MUST NOT share a name.
> On a notify block `fileName` requires an `attach` - a message with no attachment has no file to
> name. Where the attached document is rendered once for a whole fan-out, only fields of that record
> are readable, a relation hop MUST be rejected.
> Absent a pattern, an implementation supplies a default, and the SAME default for both renders; a
> document's own number is used where it declares one.

**Anchor:** Appendix A: DSL index

| [`entities.fileName`](#naming-the-rendered-file--filename) | the name a snapshot's minted copies are stored under |
| [`notify.fileName`](#naming-the-rendered-file--filename) | the name the attached render arrives under |
