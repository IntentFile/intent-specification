# One translation per record and language, in a lower-case code

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8794
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8796

## The problem

A `multilingual: true` entity keeps its per-language values in a translation table, and three things
read it: every read of the entity, a search typed in the reader's language, and a report column bound
to a translatable property. All three assume the same two things about the table: a translation's
language is the short lower-case code a translation seed declares (`bg`), and a record has at most one
translation per language. Nothing makes either true. A translation seed cannot break them, since its
code is checked, but a translation loaded any other way can: a hand-written import, a migration, a
statement run by an operator.

When one is broken, the readers disagree. A row stored as `BG` is found by an implementation that
compares codes loosely and missed by one that compares them exactly, so the form shows the
translation while the report beside it shows the stored value. Two rows for the same record and
language are worse: one reader picks one of them, another counts the record twice, and a report's
totals double. No error is raised anywhere, and the defect stays until someone compares two screens.

## The proposed shape

No new key. The rule belongs to `multilingual: true`:

```yaml
entities:
  - name: UoM
    kind: setting
    multilingual: true
    fields:
      - { name: id,   type: integer, primaryKey: true, generated: true }
      - { name: name, type: string,  required: true, length: 100 }
```

## Expected behaviour

- A record of a multilingual entity has at most one translation per language.
- A translation's language is stored as the lower-case code a translation seed declares.
- Both are declared on the translation table itself, so they hold for every writer, not only for
  translation seeds: a translation that breaks either is refused where it is written.

## Edge rules

- An application deployed before the rule may already hold translations that break it. Upgrading it
  MUST NOT fail on them and MUST NOT change them. The rule is enforced once the stored translations
  comply, and what keeps it from being enforced is reported until then.
- The rule constrains the stored code only. How a reader's requested language is reduced to that code
  (`bg-BG` is read as `bg`) is unchanged.

## Prior art / workarounds

Before this, every writer outside the seeds had to keep the codes lower-case and guard against a
second translation by hand. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8796.

## Specification text

**Anchor:** Multilingual data > Data, appended after the Normative block on report columns.

A record has at most one translation per language, and the language is stored as its lower-case code,
the one a translation seed declares. Every read of the entity, a search in the reader's language and a
report all rely on it, so it holds for whatever writes the translations, not only for the seeds.

> **Normative.**
> A conforming generator MUST declare on the translation storage of a `multilingual: true` entity that
> a record has at most one translation per language and that a translation's language is stored as a
> lower-case code, so that the storage refuses a translation breaking either, whichever writer supplies
> it. An application deployed before this rule MUST NOT fail to upgrade because its stored
> translations break it, and its stored translations MUST NOT be changed by the upgrade: the rule is
> enforced once they comply, and until then the reason it is not enforced MUST be reported.

## DSL index

No new row. The `multilingual: true` row is unchanged.
