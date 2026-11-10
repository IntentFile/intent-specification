# A boolean reads as words, and may name its own - `labels:` on a boolean field

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8564
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/PLATFORM_PR

## The problem

The format says nothing about how a `boolean` value is shown. A generator is free to print the raw
`true` / `false`, in English, whatever language the reader uses - and one did, in its list cells and
its record sheet, while its filter and its form already said Yes / No in the reader's language. And
every boolean reads the same two words, while many fields are not a yes/no question at all: an
`active` flag reads Active / Inactive, a `base` rate Base / Additional, a `paid` flag Paid / Open.
The only correction was a hand-edit of generated code or a generated catalogue, undone by the next
generation.

## The proposed shape

```yaml
- name: Product
  fields:
    - { name: id, type: integer, primaryKey: true, generated: true }
    - name: active
      type: boolean
      labels: { true: Active, false: Inactive }   # its own words
    - { name: archived, type: boolean }            # the shared Yes / No
```

## Expected behaviour

- A boolean value is never shown as `true` / `false`. Wherever a value is shown - a list cell, a
  record's details, a filter's options and its applied-filter text, a form's value options, an
  export - it reads as words.
- A field declaring `labels:` reads as its own words; a field without them reads as the
  application's shared Yes / No.
- The words are written in the default language, as `label:` is, and seed the default-language
  catalogue next to the field's label, so a language catalogue translates them like any other label.
- The order a word is resolved in: the reader's language catalogue, then the word the file
  declares, then Yes / No.
- A model without `labels:` generates as before, except that a value it used to print as
  `true` / `false` now reads as Yes / No.

## Edge rules

- `labels:` on a field that is not a `boolean` is an authoring error.
- A `labels:` without both `true` and `false` is an authoring error, and so is any other key.
- A word YAML 1.1 loads as a boolean itself (`Yes`, `No`, `On`, `Off`, `True`, `False`, unquoted)
  is an authoring error naming the key: the loaded value no longer carries the spelling. Quoted, it
  is an ordinary word.
- The words name the stored value; they are not data. Filtering and storing still use `true` /
  `false`.

## Prior art / workarounds

None; the raw value was printed, or every boolean read Yes / No. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/PLATFORM_PR.

## Specification text

**Anchor:** Data model > fields > field attributes: a table row after `countryLabels`, and the
section below appended after "label / countryLabels — a field's caption".

Table row:

| `labels` | on a `boolean` field, the words its two values read as (see [labels — a boolean's words](#labels--a-booleans-words)) |

#### labels — a boolean's words

A boolean value is shown as words, never as `true` / `false`. Without a declaration they are the
application's shared Yes / No, translated like any other label. Many fields are not a yes/no
question, and name their own:

```yaml
- { name: active, type: boolean, labels: { true: Active, false: Inactive } }
- { name: base,   type: boolean, labels: { true: Base,   false: Additional } }
```

The words are written in the default language, as `label:` is, and translated by the language
catalogues the same way.

> **Normative.**
> A conforming generator MUST NOT show a `boolean` value as its raw `true` / `false`: wherever the
> value is shown - a list cell, a record's details, a filter's options and its applied-filter text, a
> form's value options, an export - it MUST read as words. A field's `labels: { true: <word>,
> false: <word> }` gives the words; without it they are the application's shared Yes / No. The words
> MUST seed the default-language catalogue next to the field's label, and resolve from the reader's
> language catalogue, then the declared word, then Yes / No. `labels:` on a field that is not a
> `boolean`, a `labels:` missing `true` or `false` or carrying any other key, and an unquoted word
> that YAML loads as a boolean are authoring errors, reported at parse time. The words never change
> what is stored or filtered on. A field declaring no `labels:` MUST generate as before, apart from
> its values reading as Yes / No.

## DSL index

| Construct | What it does |
| --- | --- |
| [`labels` (field)](#labels--a-booleans-words) | the words a boolean field's two values read as, translated like a label |
