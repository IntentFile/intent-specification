# A catalog message picks its plural form by the count and may lower-case a label inside a sentence

- **Status:** proposed
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8777
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8847

## The problem

A refusal or warning the generated application delivers is resolved from the translation catalog
in the reader's language, with its placeholders filled after the translation is chosen. Two things a
translation needs have no form there:

1. **A word that depends on a number.** A refusal that counts records ("{count} records reference
   this customer") cannot choose the word by the count, so the default text reads "record(s)" and a
   Bulgarian catalog "запис(а)" - "1 запис(а)", "5 запис(а)".
2. **A label inside a sentence.** Entity and field labels are title-cased display names ("Currency",
   "Валута"). English keeps the capital mid-sentence; Bulgarian does not. A translator cannot say
   "this label, in lower case, here", so the label ends up quoted, or the label itself is stored in
   lower case and then reads wrongly as a column header.

The user interface's own catalogs already have both: a plural form chosen by the count, and a
lower-case format a phrase opts into. A catalog entry shared by the interface and the refusals
therefore read differently on the two sides.

## The proposed shape

No change to the file. The catalog gains two conventions, the ones the interface's catalogs use:

```json
{
  "checks": {
    "deleteReference_one": "{count} запис ({label, lowercase})",
    "deleteReference_other": "{count} записа ({label, lowercase})",
    "fieldDuplicate": "Вече съществува {entity, lowercase} с това {label, lowercase}"
  }
}
```

## Expected behaviour

- When a message's parameters carry a whole-number `count`, the entry `<key>_<category>` wins over
  `<key>`, where `<category>` is the CLDR cardinal plural category of the count in the catalog's
  language (`zero`, `one`, `two`, `few`, `many`, `other`). Without that entry the plain `<key>` is
  used, so a catalog written before keeps working.
- `{name, lowercase}` (and `{{name, lowercase}}`) inserts the value lower-cased word by word, keeping a
  word written entirely in capitals of more than one letter (an acronym such as VAT or ДДС). Only the
  phrase that opts in is affected; `{name}` inserts the value as it is.
- The generator seeds the default-language catalog with the plural forms of a counted message
  (`_one` / `_other` for English), so the default text no longer says "record(s)".

## Edge rules

- A `count` that is not a whole number selects no plural form.
- A format other than `lowercase` leaves the value as it is.
- A placeholder whose name is not among the parameters stays in the text as written, with or without
  a format.

## Prior art / workarounds

The interface's catalogs follow i18next, whose plural suffixes and formatter syntax this adopts, so
one key behaves the same in a page and in a refusal. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/issues/8777.

## Specification text

**Anchor:** Data > UI labels, appended after its paragraph.

A message delivered from the catalog follows the catalog's two conventions for words that change
with their context: a plural form chosen by the count, and a label lower-cased where it sits inside
a sentence.

> **Normative.**
> When the parameters of a catalog message carry a whole-number `count`, a conforming generator MUST
> resolve the entry `<key>_<category>`, where `<category>` is the CLDR cardinal plural category of the
> count in the catalog's language, before `<key>`, and MUST fall back to `<key>` when that entry is
> absent. It MUST insert a `{name, lowercase}` or `{{name, lowercase}}` placeholder lower-cased word by
> word, keeping a word of more than one letter that is entirely in capitals, and MUST leave a value
> whose format it does not know as it is. It MUST seed the default-language catalog with the plural
> forms of a message it delivers with a `count`.

## DSL index

| Construct | What it does |
| --- | --- |
| [catalog message `<key>_<category>`](#ui-labels) | the plural form a `count` picks, falling back to `<key>` |
| [`{name, lowercase}`](#ui-labels) | a label lower-cased inside a sentence, acronyms kept |
