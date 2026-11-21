# A restricted delete names every referencing entity, in the reader's language

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8569
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8575

## The problem

`whenTargetDeleted: restrict` refuses the delete of a record that other rows reference, "as a
validation outcome naming the referencing entity and the number of its rows". When several entities
reference the record, the rule reads as if one is enough: a product referenced by one product cost
and by two stock movements was refused naming only the product cost. The user deleted the cost,
tried again, and only then learned of the stock movements. The refusal was also written in the
default language only, whatever the reader's language, while the labels and the check messages of
the same application are translated.

## The proposed shape

No new key. The rule holds for every relation a file already declares or defaults to `restrict`:

```yaml
- name: ProductCost
  relations:
    - { name: Product, kind: manyToOne, to: Product }                          # restrict, the default
- name: StockMovement
  relations:
    - { name: Product, kind: manyToOne, to: Product, model: products }         # cross-model: the same rule
```

## Expected behaviour

- The delete of a record referenced through `restrict` relations of several entities is refused
  ONCE, and the refusal names every referencing entity with the number of its referencing rows
  (`This Product is referenced by 1 Product Cost record(s), 2 Stock Movement record(s) and cannot be
  deleted`).
- The refusal is resolved for the reader's language, as a check message is: the sentence and the
  way one referencing entity is written into it are translation entries of the deleted entity's
  model, and the entities are named by their translated labels. An entry a language does not
  translate falls back to the default-language text.
- The refusal carries its translation key and its parameters next to the resolved text, so a client
  may render it again in another language.

## Edge rules

- All references are counted in the same unit of work that refuses the delete, so the list is
  consistent with the record's state at that moment.
- A cascade that reaches a record which is itself referenced is refused with that record's own list.
- Nothing about which relations restrict changes: `nullify`, `cascade` and `keep` relations are not
  listed.

## Prior art / workarounds

None; the user found the referencing entities one refused delete at a time. Reference
implementation: https://github.com/eclipse-dirigible/dirigible/pull/8575.

## Specification text

**Anchor:** Relations & multi-model > relations > Relation attributes, the normative block of the
`whenTargetDeleted` paragraph: its first sentence is replaced by the one below; the rest of the block
is unchanged.

> **Normative.**
> A conforming generator MUST refuse the delete of a record that is referenced through relations
> declaring or defaulting to `restrict` with ONE validation outcome naming every referencing entity
> and the number of its rows, resolved for the reader's language like a check message, the
> entities named by their translated labels.
