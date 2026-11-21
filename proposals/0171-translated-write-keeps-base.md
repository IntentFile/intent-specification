# A write made in a translated language keeps the stored base value

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8735
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8844

## The problem

Every read of a translatable property of a `multilingual: true` entity is served in the caller's
requested language. The specification says nothing about the write. A form reads a record, the
reader changes one field, and the form sends back everything it read, the translated values
included. A conforming implementation could therefore store the translation as the record's own
value: a reader who changed only the price of a unit of measure would replace its stored name
("Kilogram") with the translation they were shown („Килограм“), and every reader in the default
language would see the translation from then on. Nothing reports it, and the stored value is lost.

## The proposed shape

No new key. The rule belongs to `multilingual: true`:

```yaml
entities:
  - name: UoM
    kind: setting
    multilingual: true
    fields:
      - { name: id,    type: integer, primaryKey: true, generated: true }
      - { name: name,  type: string,  required: true, length: 100 }
      - { name: price, type: decimal }
```

## Expected behaviour

- A write made in a language in which the record has a translation keeps the stored value of every
  translated property, whatever value the write carries for it.
- Every other property is written as sent.
- A write made in a language in which the record has no translation (the default language, a caller
  who requested none) writes every property as sent, as today.

## Edge rules

- The rule applies to every write that replaces the record's values, including one that merges the
  write onto a read of the record (for example a write that leaves out properties the writer may not
  see).
- Editing a translation is not part of this rule: a write never changes a translation either.

## Prior art / workarounds

Before this, an implementation that stored the payload as sent corrupted the stored value whenever a
record was saved in a translated language. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8844.

## Specification text

**Anchor:** Multilingual data > Data, appended after the Normative block on report columns.

A read in a translated language shows the translation, so what a reader sends back is the
translation, not the record's own value. A write made in that language must not store it as the
record's own value.

> **Normative.**
> A write to a `multilingual: true` entity made in a language in which the record has a translation
> MUST keep the stored value of every property that language translates, and MUST write every other
> property as sent; this holds for every write that replaces the record's values, including one merged
> onto a read of the record. A write made in a language in which the record has no translation writes
> every property as sent. A write never changes a translation.

## DSL index

No new row. The `multilingual: true` row is unchanged.
