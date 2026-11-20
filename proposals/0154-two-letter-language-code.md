# A data-language code is two lower-case letters

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8795
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8800

## The problem

A file names data languages in two places: the top-level `languages:` list (the languages the module
provides translations for) and a translation seed's `language:`. Nothing says what such a code looks
like. A reader's requested language is reduced to its primary language subtag (`bg-BG` is read as
`bg`), which is the code the stored translations are looked up by. So a code of another shape
behaves differently depending on where it ends up: a three-letter code fits nowhere in a storage
sized for the two-letter codes, and a seed that declares one is accepted by the file and fails only
when its rows are loaded. A `languages:` entry of another shape is never matched by any reader.

## The proposed shape

No new key. The codes already written keep their shape:

```yaml
languages: [en, bg]
seeds:
  - name: uoms-bg
    entity: UoM
    language: bg
    rows:
      - { id: 1, name: "Килограм" }
```

## Expected behaviour

- A data-language code is two lower-case letters, an ISO 639-1 language code.
- It is the same code everywhere it is written: a `languages:` entry and a translation seed's
  `language:`.
- Any other code is refused when the file is read, naming where it is written and the code.

## Edge rules

- A locale tag (`bg-BG`), an upper-case code (`BG`) and a three-letter code (`fil`) are all refused.
  A reader's locale tag is still reduced to its language subtag when it reads; this rule constrains
  only what a file declares.
- The language of a print template or a notification (`language:` on a print or `notify` block) is
  out of scope: it selects a template, not stored translations.

## Prior art / workarounds

Before this, a file could declare a three-letter code that a generator accepted and that failed
only when the seed was loaded into storage. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8800.

## Specification text

**Anchor:** Multilingual data > Data, appended after the paragraph on the top-level `languages:`.

A data language is named by its two-letter lower-case code, the same code everywhere: in
`languages:` and in a translation seed's `language:`. It is the code a reader's requested language is
reduced to (`bg-BG` reads as `bg`).

> **Normative.**
> A `languages:` entry and a translation seed's `language:` MUST be an ISO 639-1 language code: two
> lower-case letters. Any other value - a locale tag, an upper-case code, a three-letter code - is an
> authoring error naming where it is written and the value.

## DSL index

No new row. The `multilingual` / `languages` row is unchanged.
