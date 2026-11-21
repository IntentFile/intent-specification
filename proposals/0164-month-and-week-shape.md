# A `month` holds a month, a `week` an ISO week

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8328
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8835

## The problem

`month` (a `YYYY-MM` value) and `week` (a `YYYY-Www` ISO-week value) are stored as short strings. The
generated pickers send those shapes, but the write path stores whatever string a client sends.
`2026-5` stored beside `2026-05` opens one calendar month twice under a `unique: [project, period]`
key, and `2026-13` breaks every later reading of the value as a month. A `pattern:` cannot help: it is
a string-field attribute, and refused on these types.

## The proposed shape

No new key.

```yaml
fields:
  - { name: period, type: month, required: true }
  - { name: week,   type: week }
```

## Expected behaviour

- A `month` value is `YYYY-MM` with a month from `01` to `12`; anything else is refused as invalid
  input naming the field.
- A `week` value is `YYYY-Www` with a week from `01` to `52`, or `53` in a year that has a 53rd ISO
  week; anything else is refused the same way.
- Both refusals carry a message seeded in the translation catalog.
- A blank value on an optional field is no value, as for `pattern`.
- Nothing is normalised: `2026-5` is refused, not rewritten.

## Edge rules

- `pattern:` (and `format:`) stay refused on these types: the type fixes the shape.
- As for `pattern`, an implementation is not required to apply the check to a write that does not
  come through the generated user write surface (a create-from, a hand-written component's write).

## Prior art / workarounds

A `string` field with a `pattern:`, which loses the month picker and the month formatting.

## Specification text

**Anchor:** Entities & fields > fields > `Logical types`, after the paragraph describing `month` and
`week` and before the normative block on primary keys.

> **Normative.**
> A conforming generator MUST refuse, as invalid input naming the field and with a message seeded in
> the translation catalog, a `month` value that is not `YYYY-MM` with a month from `01` to `12`, and a
> `week` value that is not `YYYY-Www` with a week from `01` to `52`, or `53` in a year that has a 53rd
> ISO week. A blank value on an optional field MUST be read as no value. A value MUST NOT be
> normalised into the shape. The check MUST hold wherever a `pattern` is enforced. `pattern` and
> `format` on a `month` or `week` field MUST remain authoring errors.

## DSL index

No new row.
