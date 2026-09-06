# A field's label, and a label the tenant's country resolves

- **Status:** draft
- **Issue:**
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/issues/6424

## The problem

Two gaps in how a field is labelled, found while making one register work in more than one jurisdiction.

**A field cannot say what it is called.** `label:` exists on an entity, on a transition or generate button, on a report and on a dashboard widget - but not on a field. Every form caption, list column header and details-block label is therefore the humanized field name, and humanizing cannot produce an acronym, a unit or a term of art: `nationalId` renders as "National Id", `vatRate` as "Vat Rate". The only correction available is to hand-edit the generated translation catalog, which the next generation overwrites - so the model does not describe what the application shows, and the difference is re-introduced on every regeneration.

**A label cannot be keyed to the jurisdiction.** Translation catalogs are per **language**, which is right for translation and wrong for a term fixed by the company's country. A national identification number is called ЕГН in Bulgaria and Steuer-ID in Germany; which term applies depends on the company, not on the language its users happen to read the interface in. Putting the Bulgarian term in the Bulgarian catalog is wrong in both directions at once: the English-reading accountant of a Bulgarian company sees the generic "National ID" on the same record where their Bulgarian-reading colleague sees the correct local term, and a Bulgarian-reading user of a German company sees the Bulgarian term for a German field. Reversing the assignment moves the error, it does not remove it.

The information needed to resolve this is not in the model at all - it is the tenant's own country, which the platform knows. What is missing is a way to write down that a label follows it.

## The proposed shape

A field may declare its display label, and label variants keyed by country:

```yaml
entities:
  - name: Employee
    fields:
      - { name: id, type: integer, primaryKey: true, generated: true }
      - name: nationalId
        type: string
        label: National ID          # what humanizing the name cannot produce
        countryLabels:
          BG: ЕГН                   # the term a Bulgarian company uses
          DE: Steuer-ID             # the term a German one uses
      - { name: iban, type: string, label: IBAN }
```

`countryLabels` keys are ISO 3166-1 alpha-2 country codes. The country a deployment (or a tenant of it) is in is a platform-level attribute, not part of the intent: an application does not declare which country it runs in, it declares which terms apply where.

## Expected behaviour

`label:` replaces the humanized field name everywhere the field is rendered - the form caption, the list column header, the read-only details block - and seeds the field's entry in the generated default-language catalog, so it is translated exactly like any other label.

A `countryLabels` variant applies when the tenant's country matches its key, and it overrides the label **in every language**: a label resolved by country is not a translation, and treating it as one is precisely the error above. A country that declares no variant falls back to `label:`, and a field that declares no label falls back to the humanized name - so a model that declares neither behaves exactly as before.

## Edge rules

- A `countryLabels` key must be a country. A code that is no country - notably a language code - can never match a tenant, so the label would silently stay the base one forever; it is an authoring error rather than a variant that is accepted and never used.
- A variant may be declared without a base `label:`; the humanized name is then the fallback for every other country.
- A blank `label:`, and a variant with no label, are authoring errors: both read as "the label is deliberately empty", which no surface can render.
- Country resolution is a property of the tenant, never of the record. A value that varies per row - the identifier type of the particular employee - is data, and belongs in a field.
- Language and country are independent. Declaring a variant does not translate it: a country whose users read the interface in two languages sees the same country-resolved term in both, and the surrounding chrome stays translated.
- The labels of a report's columns are out of scope. A report column's label is also the name its query gives the column and the key a print template interpolates, so relabelling one there is a different change with different consequences.

## Prior art / workarounds

Today the base label can only be corrected by editing the generated catalog by hand - undone by the next generation - or by renaming the field itself so that humanizing it happens to produce the wanted text, which distorts the model to serve a caption. The country-scoped term has no workaround at all inside the model: it is either placed in a language catalog (wrong for every user whose language and company disagree) or the field is labelled generically and the local term lives in user documentation.

## Specification text

**Anchor:** Entities > Field attributes (after the attribute table)

A field's caption is the humanized form of its name. Where that is not what the field is called - an acronym, a unit, a term of art - declare it:

```yaml
- { name: nationalId, type: string, label: National ID }
```

Some terms are fixed by the **company's country** rather than by the language its users read. Such a label is declared per country, and the tenant's country resolves it:

```yaml
- name: nationalId
  type: string
  label: National ID
  countryLabels:
    BG: ЕГН
    DE: Steuer-ID
```

> **Normative.**
> A field's `label:` replaces the humanized field name wherever the field is rendered and seeds its entry in the default-language catalog, so it is translated like any other label. `countryLabels:` declares label variants keyed by ISO 3166-1 alpha-2 country code, resolved from the tenant's country and applied in **every** language - a label resolved by country is not a translation. A country that declares no variant falls back to `label:`, and a field that declares no label falls back to the humanized name. A `countryLabels` key that is not a country is an authoring error, because it can never match a tenant and would leave the base label rendering with nothing reported; so are a blank `label:` and a variant with no label.

## DSL index

| Construct | What it does |
| --- | --- |
| [`label:` (field)](#field-attributes) | the field's display label, replacing the humanized name and seeding its catalog entry |
| [`countryLabels:`](#field-attributes) | label variants resolved from the tenant's country rather than the reader's language |
