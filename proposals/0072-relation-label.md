# A to-one relation and a subset take `label` and `countryLabels`

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/108, https://github.com/eclipse-dirigible/dirigible/issues/7650
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7676
- **Amends:** [0031 - a field's label and country variants](0031-field-label-and-country-variants.md)

## The problem

0031 gives a **field** a caption and country variants; the relation next to it has neither. A relation
carries the target's name, so a company's `Year.company` renders its picker, its list column and its
details row as "Company", and there is no way to say "Issuing company" in the file. The caption a
relation needs is as often a term of art as a field's: "Debtor", "Payer", "Cost centre", "Issuing
company", "Ship-to". The only place left for it is the generated catalog, which the next generation
overwrites.

## The proposed shape

```yaml
relations:
  - { name: company, kind: manyToOne, to: Company,
      label: Issuing company, countryLabels: { BG: Издаваща фирма } }
  - { name: vatScheme, kind: subset, of: TaxRate, label: VAT scheme }
```

## Expected behaviour

Both keys are accepted on a to-one relation (`manyToOne`, `oneToOne`) and on a `subset`, with
exactly a field's rules: the label replaces the humanized relation name wherever the relation is
rendered - picker caption, list column header, details row - and seeds the default-language catalog
entry; `countryLabels` are variants keyed by ISO 3166-1 alpha-2 country, resolved from the tenant's
country in every language.

## Edge rules

- A blank label and a key that is not an alpha-2 country are authoring errors, as for a field.
- A `oneToMany` renders no control of its own - the key lives on the child - so a label authored
  there is refused with a message naming the construct that does render one; it is never silently
  dropped.
- On an n:m relation the keys apply to the picker side of the materialised link entity (0019).
- An unlabelled relation generates exactly as before.

## Prior art / workarounds

A hand-edited translation catalog. Reference implementation: https://github.com/eclipse-dirigible/dirigible/pull/7676.

## Specification text

**Anchor:** Entities & fields > label / countryLabels - a field's caption, appended; and a row in the
relation attribute table.

A to-one relation and a subset take the same two keys, with the same rules, since a relation's caption
is as often a term of art as a field's:

```yaml
- { name: company, kind: manyToOne, to: Company, label: Issuing company, countryLabels: { BG: Издаваща фирма } }
```

> **Normative.**
> `label:` and `countryLabels:` MUST be accepted on a `manyToOne`, a `oneToOne` and a `subset`
> relation and MUST have the effect they have on a field: the label replaces the humanized relation
> name wherever the relation is rendered and seeds the default-language catalog; the country
> variants are resolved from the tenant's country in every language. The validation is a field's: a
> blank label and a key that is not an ISO 3166-1 alpha-2 code are authoring errors. Either key on
> a `oneToMany` MUST be reported as an authoring error naming the construct that renders a control;
> it MUST NOT be dropped. On a many-to-many relation the keys apply to the picker side of the link
> entity. A relation declaring neither MUST generate byte-identical output.

## DSL index

| Construct | What it does |
| --- | --- |
| [`label` / `countryLabels` (relation)](#label--countrylabels--a-fields-caption) | a to-one's or a subset's caption, and its country-resolved variants, with a field's rules |
