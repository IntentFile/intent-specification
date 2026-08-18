# One document, many recipients: `attach: recordPrint` and the fan-out scoping rule


- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/6717
- **Discussion:** https://github.com/IntentFile/intent-specification/pull/20

Closes the "Planned" entry that named this gap.

`forEach` + `attach: print` attaches **each row's own** document, which is what a per-row document (a payslip) needs. The mirror shape had no syntax: **one** document sent to **many** recipients, where the rows ARE the recipients and the attachment belongs to the record they hang off - a request for quotation mailed to each invited supplier, an agenda mailed to each participant.

**`attach: recordPrint`** renders the fan-out's **anchor record**: fan-out-only (outside one, `attach: print` already renders that record), the anchor is what must satisfy the document requirement, `language` / `languageFrom` select ITS render language, and a conforming generator MUST render the document **once** per fan-out rather than once per recipient.

The section also settles the point the gap entry left open - **an explicit scoping rule when both records are addressable**: a bare path resolves against the ROW, the reserved prefix `record.` addresses the anchor (one field of it), the recipient may never be record-scoped, and `record.` outside a fan-out is rejected. Implicit mixing is how a message quotes the wrong party, and nothing in the rendered text would show it.

Also normative: a `forEach` on a `schedules[].notify` or a `notifications[]` entry MUST be rejected - the first already runs once per matched row, the second is about the event record, so an accepted declaration there changes nothing and sends a different message than the one written down.

Appendix A gains the `attach: recordPrint` row.

Proven out by the reference implementation: eclipse-dirigible/dirigible#6740.

## Specification text

The prose below is what a release folds into the next version document, at the anchors
given. It was written against `versions/1.2.md` and is carried here unchanged.

**Anchor:** Declarative glue > The notify block — and `attach: print`, sending the document itself

> **Normative.** `attach` is `print` - the record the block is about - or, inside a fan-out, [`recordPrint`](#one-document-many-recipients-attach-recordprint). A block declaring `print` MUST be about an entity with a **line-items child** — the shape a print template is generated for; a generator MUST reject `attach: print` on any other entity rather than send a message without the document it promised. `language` names one of the document's print-template languages; when omitted, the default language is used. The attachment MUST be produced from the record's own data through the same path the interactive print takes, so a document mailed and a document printed are the same document.

**Anchor:** Declarative glue > The notify block — and `attach: print`, sending the document itself > One message per related row: `forEach`


> **Normative.** `forEach` is authored on a `transitions[].notify` or a `serviceTask`'s `args.notify`.
> A `schedules[].notify` already runs once per matched row and a `notifications[]` entry is about the
> event record, so a `forEach` on either MUST be rejected rather than ignored - an accepted declaration
> that changes nothing sends a different message than the one that was written down.

#### One document, many recipients: `attach: recordPrint`

The mirror shape: the related rows are only the **recipient list** and the document belongs to the
record they hang off - a request for quotation mailed to each invited supplier, an agenda mailed to
each participant. `attach: print` cannot express it (it renders the row, which is nobody's document);
`attach: recordPrint` renders the fan-out's **anchor record** - the record the block is about - once,
for everybody.

```yaml
    notify:
      forEach: InvitedSupplier                      # the rows: the recipient list
      to: Supplier.email                            # the ROW's supplier - the rows ARE the recipients
      subject: "RFQ {record.number}"                # {record.<field>} = the ANCHOR RECORD's field
      body: "Dear {Supplier.name}, please quote by {record.deadline}."   # bare = the ROW
      attach: recordPrint                           # the RECORD's document, rendered once
```

> **Normative.** `attach: recordPrint` is only meaningful inside a fan-out and MUST be rejected without
> one - outside a fan-out `attach: print` already renders that very record. The **anchor record** MUST
> satisfy the document requirement; the row need not. `language` / `languageFrom` then select the
> anchor's render language, read off the anchor, because there is exactly one render: a conforming
> generator MUST render the document ONCE per fan-out and attach the same result to every message,
> never once per recipient.

> **Normative.** Inside a fan-out a **bare** path - the recipient, `{field}`, `{Relation.field}` -
> resolves against the **ROW**, and the reserved prefix **`record.`** is the only way to address the
> anchor record: `{record.<field>}` names ONE field of it, and a longer path MUST be rejected. The
> recipient MUST NOT be record-scoped: the rows are the recipients, so a record-scoped address would
> send the same message to the same address once per row. `record.` outside a fan-out MUST be rejected
> too, since there every bare path already resolves against the record. Which record a path reads is
> therefore always written down and never inferred - nothing in a rendered message reveals that the
> wrong one was read.

**Anchor:** Appendix A: DSL index

| [`notify.forEach`](#one-message-per-related-row-foreach) | fan the block out over a related collection: one message per row, every bare path resolved against the row |
| [`attach: recordPrint`](#one-document-many-recipients-attach-recordprint) | in a fan-out: attach the ANCHOR record's document, rendered once, to every recipient's message (`{record.<field>}` addresses that record) |
