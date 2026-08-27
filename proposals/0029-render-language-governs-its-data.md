# A render's language governs its data, not only its template

- **Status:** draft
- **Issue:**
- **Reference implementation:** https://github.com/eclipse-dirigible/dirigible/issues/6945, https://github.com/eclipse-dirigible/dirigible/issues/6947

## Why

Two rules this format already states never meet, and the gap between them puts one document in two
languages.

- **Multilingual data** says every read of a translatable property is served **in
  the caller's requested language**.
- A render's language declaration - `language:` / `languageFrom:` on a notify block, the language a
  versioned copy is minted in, the language a reader picks in the print action - selects **the
  template**.

A render is not a read by a caller. Nothing joins the two, so an implementation is free to answer the
question "which language is the data in?" differently at every render, and in practice does:

- the **interactive print** runs as a caller, so the values come out in the language the reader
  happens to be *browsing* in - not the one they just chose in the print dialog;
- a **minted copy** and a **mailed attachment** are produced by a workflow step, a schedule or an
  event handler. There is no caller to ask, so the translated read has no requested language and
  falls back to the stored values.

The result is a Bulgarian sales invoice whose template reads `ФАКТУРА` and `ПАДЕЖ` over data reading
`Bank transfer`, `E-mail` and `ISSUED`. Every translatable value in it - the payment method, the
delivery method, the status, any nomenclature the document reaches - is in the wrong language, while
everything the author wrote by hand is in the right one.

It is worst exactly where it matters most. The interactive print is looked at by the person who asked
for it, who can see the problem. The other two renders are the ones nobody re-reads before they
leave: the **archived immutable copy** an audit is answered from, and the copy the **counterparty**
receives with no application around it. A document that names its own language in its heading and
then contradicts it in its body is not a translation problem; it is a document that cannot be relied
on.

## What this adds

No new key. Every one of these renders already names a language - the declaration is there, and this
proposal says what it governs.

```yaml
entities:
  - name: SalesInvoiceCopy
    function: Snapshot
    languageFrom: customer.language      # the copy is minted in the customer's language
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
      languageFrom: customer.language    # the attachment is rendered in the customer's language
```

The rule: **the language a render declares is the requested language for every read performed to
produce that render.** The template and the data are then one decision, made once, in one place -
which is what an author writing `languageFrom: customer.language` already believes they wrote.

## Expected behaviour

A conforming generator produces a rendered document whose translatable values are resolved in the
render's own language, at all three call sites:

| Render | Its language | What must read in it |
| --- | --- | --- |
| the interactive print | the language chosen in the print action | the document's own data and everything it reaches |
| a versioned copy (`function: Snapshot`) | the copy's mint language | the same |
| a notify `attach` | the block's `language:` / `languageFrom:` | the attached document, or the attached report's rows |

The existing rule that an attachment "comes from the record's own data through the same path the
interactive print takes, so a document mailed and a document printed are the same document" is what
this makes true: today the two paths differ precisely in the language of their data.

## Edge rules

- **Fallback is unchanged and per property.** A property with no translation in the render language
  reads its stored value, exactly as any read does. A render language the module provides no
  translations for is therefore not an error - it produces a document in the default language, which
  is what the fallback is for.
- **Matching is unaffected**, for the same reason it is unaffected for a report: what a render
  *selects* - a print template's row filter, an attached report's `filter:` and `scope:` - is
  evaluated against the stored values. Choosing a language can change how a document reads, never
  which rows it contains.
- **A fan-out renders once, in one language.** Where the anchor's document is attached for every
  recipient, the render language is read off the anchor and there is one render, so there is one data
  language too. A per-row `attach` renders per row, and each row's render uses its own resolved
  language.
- **Nothing leaks between renders.** A render's language applies to that render and ends with it: two
  documents minted one after another, in different languages, are each in their own.
- **No new rejection.** This constrains no new authoring; it removes a freedom implementations had.

## Prior art / workarounds

There is no workaround in the format. An author who needs a document to be internally consistent
today has to avoid multilingual nomenclature in printed documents altogether - i.e. avoid the
construct in the one place it is most visible - or accept that the archived copy and the mailed copy
say something different from the screen.

Implementations resolved the unstated question in two different ways at two call sites of the same
feature, which is how the defect was found: the interactive path used the reader's browsing language
and the server-side path used none at all. That is the signature of a rule the format states only
half of.

## Notes

Deliberately **not** part of this proposal:

- **The language of the message's own text.** Whether a notify block's `language:` also governs the
  translatable values interpolated into its `subject` and `body` is a separate question with a
  different answer: an attachment is a contractual document and belongs in the document's language,
  while the covering message belongs in the **recipient's**. Conflating them here would settle by
  accident something that deserves its own proposal.
- **A UI reading in a language other than the caller's.** The screen has a caller and the existing
  rule already serves it correctly. This is about renders, which do not.
- **How an implementation carries the language into the read.** Observable behaviour only; the
  mechanism is an implementation's business.

## Specification text

The prose below is what a release folds into the next version document, at the anchors given.

**Anchor:** Multilingual data > Data (after the existing Normative block)

A **render** - the interactive print, a versioned copy, a document or report attached to a message -
has a language of its own, chosen in the print action or declared by `language:` / `languageFrom:`.
That language selects the template, and it is also the language the render's data is read in: a
document names its language in its heading and must not contradict it in its body.

> **Normative.**
> The language a render is produced in MUST be the requested language for every read performed to
> produce it, including reads made where there is no caller to ask - a copy minted by a workflow, a
> document or report attached by a scheduled or event-driven message. A render MUST NOT resolve its
> data in a language other than the one its template was selected in.
> The per-property fallback is unchanged: a property with no translation in the render's language
> reads its stored value, so a render language the model provides no translations for produces a
> document in the default language rather than an error.
> What a render **selects** MUST remain evaluated against the stored, untranslated values - a print
> template's row filter and an attached report's `filter:` and `scope:` alike - so a language can
> change how a rendered document reads but never what it contains.
> A render's language MUST apply to that render alone: consecutive renders in different languages
> MUST each be produced in their own.

**Anchor:** Printable documents (after "To add a language, add a file under a sibling language folder")

The chosen language selects the template **and** the language the document's data is read in, so a
document rendered in one language is in that language throughout - see
[multilingual data](#multilingual-data). The same holds wherever a render's language is declared
rather than chosen: the language a [versioned copy](#attachments-and-snapshots) is minted in, and the
`language:` / `languageFrom:` of a
[notify block that attaches one](#the-notify-block--and-attach-print-sending-the-document-itself).
