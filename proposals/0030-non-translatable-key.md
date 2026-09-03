# A key of a multilingual entity is not translated

- **Status:** draft
- **Issue:**
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/issues/6545

## The problem

Marking an entity `multilingual: true` makes **every** string property translatable — translatability is derived from the type, and nothing in the model can say otherwise. That is right for a label and wrong for a **key**.

Some strings are matched, not read. A determination rule is selected by a literal the model carries:

```yaml
postings:
  - name: salesInvoicePosting
    rule: { entity: PostingRule, match: { documentType: "Sales Invoice" } }
```

An inbound arrival resolves a relation from a business key the sender sends:

```yaml
inbound:
  - name: partnerFeed
    path: /partners
    create: Partner
    map:
      Country: { lookup: Country, by: code, from: countryCode }
```

Both compare a value against a stored column. If that column is translatable, the comparison is on a moving target: the read overlay shows the translated value, saving the row from the UI puts the translated value into the stored column, and from then on the authored literal — or the sender's key — matches nothing. There is no error. The posting simply never fires again for that tenant; the arrival's lookup resolves nothing. A rule that has silently stopped applying is indistinguishable from a document nobody posted.

The model has the information to prevent this — it knows which properties are matched — but no way to write it down.

## The proposed shape

A field of a multilingual entity may declare that it carries no translation:

```yaml
entities:
  - name: PostingRule
    kind: setting
    multilingual: true
    fields:
      - { name: id,           type: integer, primaryKey: true, generated: true }
      - { name: name,         type: string }                              # a label - translated
      - { name: documentType, type: string, translatable: false }         # a key - never translated

  - name: Country
    kind: setting
    multilingual: true
    fields:
      - { name: id,   type: integer, primaryKey: true, generated: true }
      - { name: name, type: string }
      - { name: code, type: string, unique: true, translatable: false }   # the business key
```

## Expected behaviour

A property marked `translatable: false` has no per-language value anywhere: it gets no column in the entity's translation table, it is never overlaid on a read, a report column bound to it reads the stored value, and a translation seed cannot carry it. Every other property of the entity is unaffected — the marker is per field, not per entity.

The default is `true`, so an existing model behaves exactly as before and generates identical output.

Because the property is now declarable, the two sites that match on a stored string can be held to it: a determination rule's `match` column and a business-key lookup's `by` field must not be translated, and an intent that matches on a translated property is an authoring error rather than a rule that quietly stops applying.

## Edge rules

- `translatable: false` is only meaningful where a translation could otherwise exist: on an entity that is not `multilingual: true` there is no translation table to be kept out of, and on a non-string property there is no translated value at all. Both are authoring errors — the alternative is a marker that is accepted and ignored, which reads as protection that is not there.
- `translatable: true` is the default and declaring it explicitly changes nothing.
- A translation seed whose rows set a property marked `translatable: false` is an authoring error: the property has no place in the translation table, so the seed describes data that cannot be stored.
- A property computed on write is not translatable either, for the same reason it never was: its value is produced on every write, so a per-language variant of it has no meaning.
- The primary key is not translatable (it identifies the row), and marking it is redundant rather than an error.
- A determination rule's `match` column, and a business-key lookup's `by` field, must not be a translated property of a multilingual entity. The authoring error names the marker, since marking the property as the key it is, is the fix.
- The marker constrains storage, not authoring convenience: a key that must be shown differently per language is a key with a separate label property beside it, not a translated key.

## Prior art / workarounds

Today the only way to keep a key out of the translation table is to keep the entity out of it — dropping `multilingual: true` from a nomenclature that legitimately needs translated labels — or to hand-write the matching so it reads the stored value instead of the translated one, which leaves the model no longer describing what the application does. The third and most common outcome is that nobody notices until a rule stops applying in one tenant's language and the difference has to be found by reading data.

## Specification text

**Anchor:** Multilingual data > Data (after the report normative block)

Some string properties are **keys, not labels** — the column a [posting](#postings--source-document-to-ledger)'s determination rule matches on, the business key an [arrival](#inbound--arrivals-from-outside)'s lookup resolves a relation by, a code another model refers to. Such a value identifies a row; it is not text to be read in the reader's language. Declare it `translatable: false`:

```yaml
entities:
  - name: PostingRule
    kind: setting
    multilingual: true
    fields:
      - { name: id,           type: integer, primaryKey: true, generated: true }
      - { name: name,         type: string }                       # a label - translated
      - { name: documentType, type: string, translatable: false }  # a key - never translated
```

> **Normative.**
> A property marked `translatable: false` carries no per-language value: it has no column in the entity's translation table, it is never overlaid on a read, a report column bound to it reads the stored value, and a translation seed that sets it is an authoring error. The default is `true`, and declaring it explicitly changes nothing. The marker must be able to mean something: on an entity that is not `multilingual: true`, or on a property that is not string-typed, it is an authoring error rather than an accepted no-op. A property a determination rule matches on, and a property a business-key lookup resolves by, must not be translated — an intent that matches on a translated property is an authoring error naming this marker, because a translated key stops matching the value the intent was authored with and does so with no observable failure.

## DSL index

| Construct | What it does |
| --- | --- |
| [`translatable: false`](#multilingual-data) | keeps a key of a multilingual entity out of the translation table, so what a rule matches on cannot be translated out from under it |
