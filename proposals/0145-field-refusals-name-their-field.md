# A refusal about one field is in the reader's language and names the field

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8596, https://github.com/eclipse-dirigible/dirigible/issues/8597
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8624

## The problem

A check message, an authored business-key message and a delete refusal are delivered in the
reader's language. The refusals a generated application makes on its own were not:

- a value repeating a field declared `unique: true` (`A UoM with this 'Sap' already exists`);
- a required value left out (`The 'Numerator' property is required`);
- a text longer than its field's `length`, and a value that does not match its `pattern`.

They were fixed sentences in the default language, naming the field by its generated property name
rather than its label. A reader of another language met both languages in one form, depending on
which rule refused the save.

The form marked the offending field by reading the property name out of that English sentence. So
translating the sentence alone would stop the field from being marked. A refusal of a `checks:` rule
about one field (`compare`, `requiredWhen`) was never marked at all, because its authored sentence
need not quote the field. The person read the reason in a banner and had to find the field
themselves.

## The proposed shape

No new key. The rule holds for every field and check a file already declares:

```yaml
- name: Uom
  checks:
    - { kind: compare, field: numerator, op: gt, value: 0, message: "The numerator is greater than zero" }
  fields:
    - { name: sap,       type: string, length: 6, required: true, unique: true, label: SAP Code }
    - { name: numerator, type: decimal }
```

## Expected behaviour

- The refusals of a required value, a too long value, a value off its pattern and a repeated
  `unique: true` value are translation entries of the entity's model, like a check message. Each
  names the field, and the repeated value also names the entity, by its translated label. An entry
  a language does not translate falls back to the default-language text.
- Every refusal about one field of the record names that field in a structured form, beside the
  message and its translation key. This covers the four refusals above and a check whose `field:` is
  one of the record's own fields (`compare`, `requiredWhen`). A client marks the field from that,
  whatever language the message is in.
- A check about no single field (`forbidWhen`, `agree`, `exists`, `exactlyOne`, the document
  checks), and one whose `field:` reaches a related record, names none. Its message is shown as a
  whole, as before.

## Edge rules

- A field the form does not show (a hidden or calculated one) is not marked. The message is shown
  as a whole.
- An authored business-key message on a composite `unique:` keeps its own text and key. It names no
  single field.
- A soft check (`severity: warn`) refuses nothing. It is unchanged.

## Prior art / workarounds

Reading the field from the message text, which works only in the default language and only for a
message that quotes the property name. Reference implementation: https://github.com/eclipse-dirigible/dirigible/pull/8624.

## Specification text

**Anchor:** Entities & fields > checks — declarative validations, the normative block about the
translation catalog: the sentences below are appended to it.

> **Normative.**
> A conforming generator MUST also seed the default-language catalog with the refusals it makes on
> its own - a required value left out, a value longer than its `length`, a value off its `pattern`, a
> value repeating a `unique: true` field - and MUST deliver them in the reader's language, naming the
> field and the entity by their translated labels. A refusal about one field of the record - one of
> those, or a check whose `field:` is a field of the record itself - MUST name that field in a
> structured form beside the message, so that a client can mark the field without reading it out of
> the message.
