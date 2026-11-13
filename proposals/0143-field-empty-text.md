# What an empty value reads as - `emptyText:` on a field

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8605
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8606

## The problem

The format says nothing about how a field with no value is shown, and for many fields "no value"
means something. A member's loan limit that is empty means *no limit*; a membership that never
expires has no expiry date; a nickname nobody chose is *not set*. A generator shows such a value as
a placeholder mark, or leaves the field out of a details block that shows only filled values - and a
reader cannot tell "no limit" from "not loaded" or "not filled in yet". The only workaround is a
sentinel value (`-1`, `9999-12-31`), which every rule, check and report then has to know about.

## The proposed shape

```yaml
- name: Member
  fields:
    - { name: id,        type: integer, primaryKey: true, generated: true }
    - { name: name,      type: string, length: 80, required: true }
    - { name: maxLoans,  type: integer, readOnly: true, label: Loan limit, emptyText: No limit }
    - { name: expiresOn, type: date, label: Membership expires, emptyText: Never }
    - { name: nickname,  type: string, length: 40, emptyText: Not set }
```

## Expected behaviour

- Wherever the field's value is shown as a value - a record's page and its details, a list cell, a
  list's record sheet, a collection panel or related-records register, a record's subtitle or
  summary line - an empty value (none, a blank text, an empty list) reads as the declared text instead
  of the generator's placeholder mark.
- A details block that shows only filled values always shows the row of a field declaring
  `emptyText:`, so the meaning of its emptiness is never hidden.
- On the create and edit forms the text is the placeholder of the field's free-text or number input,
  telling the editor what leaving it empty means. It is never stored.
- The text is written in the default language, as `label:` is, and seeds the default-language
  catalogue next to the field's label, so a language catalogue translates it like any other label.
- Nothing about the data changes: storage, the REST answers, exports and prints carry the empty value
  as before.
- A model without `emptyText:` generates exactly as before.

## Edge rules

- A blank `emptyText:` is an authoring error (remove it to keep the default rendering).
- `emptyText:` is an authoring error on a field that is never empty or never shown as a value: a
  `boolean` (it always reads as one of its two words), a `uuid` (issued on create), the primary key,
  a `required` field (an empty value is refused), a document `number:` (an unissued number has its own
  marker), a field with `placement: hidden`, an `aggregate` field, the document title and a chat
  message body.
- It is not a relation attribute; on a relation it is an unknown key.
- A field a reader may not see (`visibleTo:`) shows that reader neither its value nor its text.

## Prior art / workarounds

A sentinel value, or a hand-edit of generated pages undone by the next generation. Reference
implementation: https://github.com/eclipse-dirigible/dirigible/pull/8606.

## Specification text

**Anchor:** Data model > fields > field attributes: a table row after `countryLabels`, and the
section below appended after "label / countryLabels — a field's caption".

Table row:

| `emptyText` | the text an empty value reads as wherever the field is shown (see [emptyText — what an empty value reads as](#emptytext--what-an-empty-value-reads-as)) |

#### emptyText — what an empty value reads as

For many fields an empty value means something: no loan limit is *no limit*, no expiry date is
*never*. `emptyText:` says so, instead of a sentinel value every rule and report would have to know:

```yaml
- { name: maxLoans,  type: integer, readOnly: true, label: Loan limit, emptyText: No limit }
- { name: expiresOn, type: date, label: Membership expires, emptyText: Never }
```

The text is written in the default language, as `label:` is, and translated by the language
catalogues the same way.

> **Normative.**
> Wherever a conforming generator shows a field's value as a value - a record's page and its
> details, a list cell, a list's record sheet, a collection panel or related-records register, a
> record's subtitle or summary line - an empty value (none, a blank text, an empty list) of a field
> declaring `emptyText: <text>` MUST read as that text. A details block that otherwise shows only
> filled values MUST show the row of such a field. The text MUST be the placeholder of the field's
> free-text or number input on the create and edit forms, and MUST NOT be stored. It MUST seed the
> default-language catalogue next to the field's label and resolve from the reader's language
> catalogue, then the declared text. A field the reader may not see MUST NOT show its text. The text
> MUST NOT change what is stored, served, exported or printed. A blank `emptyText:`, and
> `emptyText:` on a `boolean`, a `uuid`, the primary key, a `required` field, a document `number:`, a
> `placement: hidden` field, an `aggregate` field, the document title or a chat message body, are
> authoring errors reported at parse time. A model declaring no `emptyText:` MUST generate as
> before.

## DSL index

| Construct | What it does |
| --- | --- |
| [`emptyText` (field)](#emptytext--what-an-empty-value-reads-as) | the text a field's empty value reads as wherever it is shown, and its input's placeholder |
