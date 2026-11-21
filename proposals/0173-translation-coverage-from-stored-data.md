# Missing data translations are counted from the stored translations, not from `languages:`

- **Status:** proposed
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8789
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8848

## The problem

The top-level `languages:` declares which languages a module provides translations for. An
implementation that tells its users what a module leaves untranslated used that declaration as the
answer for the module's data:

- a module that declares `languages: [en, bg]` was never reported, even when its translation table
  held no `bg` row at all, or when every record added after the seed had none - nothing writes a
  translation while the application runs, so the table only holds what a seed or a hand-written
  statement put there;
- a module whose every record is translated, but which declares no `languages:`, was reported as
  untranslated.

The declaration says what the author intended; it says nothing about what the data holds.

## The proposed shape

No new key. The report counts what the translation tables hold:

```yaml
languages: [en, bg]
entities:
  - name: Currency
    multilingual: true
    fields:
      - { name: id,   type: integer, primaryKey: true, generated: true }
      - { name: name, type: string,  required: true, length: 100 }
```

With 160 currencies, three of them without a `bg` translation, the report reads "Currencies: 3 of
160 records have no Bulgarian translation" - whatever `languages:` declares.

## Expected behaviour

- For every `multilingual: true` entity and every supported language other than the default one,
  the number of records and the number of them with no translation in that language are available
  to whoever may read the entity.
- A report of missing data translations names the entity and the counts, in the reader's language.
  An entity whose records all have a translation is not reported.
- `languages:` stays a declaration. It decides nothing a translation table can answer; it is used
  only for a module whose stored translations cannot be read.

## Edge rules

- The default language is not counted: the records themselves hold its values.
- A record counts as translated in a language when the translation table holds a row for it in that
  language, whichever of its properties that row fills.
- A reader who may not read an entity is told nothing about it, neither counts nor the fallback.

## Prior art / workarounds

Before this, the only way to learn that a module's data was untranslated was to switch the language
and look at its lists. Reference implementation: the platform PR linked above.

## Specification text

**Anchor:** Multilingual data > Data, appended after the paragraph that starts "The set of languages
the whole stack supports".

A module's `languages:` declares its intent; what its data holds is what its translation tables
hold. Whether a multilingual entity's data is translated into a language is answered by counting its
records that have no translation row in that language.

> **Normative.**
> An implementation that reports untranslated data MUST judge each `multilingual: true` entity by its
> stored translations: per supported language other than the default one, the number of its records
> and of those with no translation in that language. It MUST NOT report an entity whose records are
> all translated because the module does not declare the language, nor leave one unreported because
> the module declares it. It MAY fall back to the `languages:` declaration only for a module whose
> stored translations cannot be read. The counts are given only to a reader who may read the entity.

## DSL index

| Construct | What it does |
| --- | --- |
| [`multilingual` / `languages`](#multilingual-data) | translation tables + read-time translation overlay; untranslated data counted from the tables |
