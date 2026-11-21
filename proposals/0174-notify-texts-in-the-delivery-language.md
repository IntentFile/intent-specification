# A message is written in the language it is delivered in

- **Status:** proposed
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8787
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8849

## The problem

A notify block's `subject:` and `body:` are one fixed text, in whatever language the author wrote
them. `language:` / `languageFrom:` reach only the attachment. A supplier whose record says they read
Bulgarian therefore receives an English mail with a Bulgarian document attached, and writing the
subject in Bulgarian only moves the problem to every English-speaking recipient. One notification
cannot serve both, although the file already says which language each recipient reads.

A notice without an attachment ("your quote was not selected", "the invoice was voided") has no way
to say its recipient's language at all: `language:` / `languageFrom:` are refused without `attach`.

[0029](0029-render-language-governs-its-data.md) left this open on purpose: "the covering message
belongs in the recipient's" language while the attachment belongs in the document's. This proposal
settles it. A file declares one language per message - `language:` / `languageFrom:` - and in
practice it is read off the counterparty the message goes to (`languageFrom: Customer.language`), so
it is the recipient's language as well as the document's. A second, message-only knob would ask the
author to say the same thing twice, and a mail whose cover and attachment disagree is the case the
problem above describes.

## The proposed shape

No new key. The texts stay as they are authored, and the language the block already declares is the
language of the whole message:

```yaml
transitions:
  - name: DeclineQuote
    forEntity: SupplierQuote
    from: [1]
    setStatus: 3
    notify:
      to: Supplier.email
      subject: "Your quote for {Rfq.number}"
      body: "Dear {Supplier.name}, we have not selected your offer this time."
      languageFrom: Supplier.language        # now allowed without attach
```

## Expected behaviour

- The authored `subject`, `body` and `html` are the default-language texts of the message. A
  generator seeds them in the default-language translation catalog, one entry per notify block, so a
  catalog of another language translates them as it translates a label.
- A message is delivered in one language: the one its attachment renders in where it carries one,
  else the block's own `language:` / `languageFrom:`, else the default language. Its subject, body and
  marked-up alternative are the catalog texts of that language where the catalog has them, and the
  authored texts otherwise.
- The `{placeholder}`s are filled after the translation is chosen, so a translation may move them.
- The language of the person or event whose action sends the message plays no part: the message is
  the recipient's.

## Edge rules

- `language:` and `languageFrom:` are accepted on a block without `attach`, with the same rules as on
  one: mutually exclusive, `languageFrom` a one-hop `relation.field` to a string field of the record
  the message is about (the row inside a `forEach`).
- With `attach: recordPrint` the anchor's document decides the language, and every row's message is
  delivered in it.
- A language whose catalog lacks an entry, or an entry of a block, falls back to the authored text of
  that entry alone.

## Notes

Deliberately **not** part of this proposal: the translatable values a `{placeholder}` interpolates
(the name of a multilingual nomenclature row) are read as before. Whether they too follow the
message's language is left to a proposal of its own.

## Prior art / workarounds

Before this, an author chose one language for every recipient of a notification, or wrote the mail by
hand in custom code that read the recipient's language itself. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/issues/8787.

## Specification text

**Anchor:** Declarative glue > The notify block — and `attach: print`, after the Normative paragraph
that requires `attach: print` to be about an entity with a line-items child. The release also
replaces, under *A render reads in its own language*, the sentence "Whether a notify block's
`language:` also governs the translatable values interpolated into its own `subject` and `body` is
deliberately not settled here: ..." with: "A notify block's `subject` and `body` are written in the
language its attachment renders in - see [the notify block](#the-notify-block--and-attach-print-sending-the-document-itself).
Whether the translatable values they interpolate follow it is not settled here."

A message is written in the language it is delivered in. The authored `subject`, `body` and `html`
are its default-language texts; the catalogs carry the others, as for a `label:`. `language:` and
`languageFrom:` name the language of the whole message - its texts and its attachment - and may be
declared on a block that attaches nothing.

> **Normative.**
> A conforming generator MUST seed the default-language catalog with every notify block's `subject`,
> `body` and `html`, keyed per block, and MUST deliver each message in one language: the language
> its attachment renders in where it has one, else the block's `language` / `languageFrom`, else the
> default language - never the language of the person or event whose action sends it. The texts MUST
> be the catalog's in that language where the catalog has an entry and the authored ones otherwise,
> with placeholders resolved after the translation is chosen. `language` / `languageFrom` on a block
> without `attach` MUST be accepted under the rules that hold with one.

## DSL index

| Construct | What it does |
| --- | --- |
| [`notify`](#the-notify-block--and-attach-print-sending-the-document-itself) | `language` / `languageFrom` - the language of the whole message, with or without an attachment |
