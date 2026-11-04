# A decimal is rounded to its scale before checks and calculations read it

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/163, https://github.com/eclipse-dirigible/dirigible/issues/8107
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8265

## The problem

A decimal field has a declared scale (`precision` / `scale`, default 16, 2), and the stored value is
rounded to it. If `checks` and calculated fields read the value as the caller sent it, they reason
about a value that is never stored:

```yaml
fields:
  - { name: hours,  type: decimal, precision: 18, scale: 2 }
  - { name: rate,   type: decimal }
  - { name: amount, type: decimal, calculatedOnCreate: "Hours * Rate", calculatedOnUpdate: "Hours * Rate" }
checks:
  - { kind: compare, field: hours, op: gt, value: 0, message: "Hours must be positive" }
```

`0.0049` hours passes `hours gt 0`, is stored as `0.00`, and still bills `Hours * Rate = 0.49` -
a zero row the intent says cannot exist, with an amount that does not follow from it. 1.11 states
the scale of the column; it does not say when, relative to checks and calculations, the value takes
that scale.

## The proposed shape

No new key. The fields and checks are the ones 1.11 already has (above).

## Expected behaviour

- A decimal is rounded half-up to its declared scale before `checks`, `calculatedOnCreate` /
  `calculatedOnUpdate` and the pre-write validation read it, on create and on update.
- So the value checked, the value calculated from and the value stored are one value: a create of
  `0.0049` hours, and an update to `0.004`, are refused by `hours gt 0` (400); `7.9949` hours at a
  rate of 100 stores `7.99` and `799.00`; an update to `2.005` hours stores `2.01` and `201.00`.
- A value already within its scale is untouched, so nothing stored or answered before changes shape.

## Edge rules

- The primary key is not concerned (it is an integer).
- Each decimal is rounded to its own declared scale (the default scale when it declares none).

## Prior art / workarounds

None within the model: the check passed, the calculation ran on the unrounded value, and the stored
row contradicted both. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8265.

## Specification text

**Anchor:** Entities & fields > Logical types, a paragraph and a normative block appended after the
existing normative block.

A `decimal` has a declared scale, and the value it holds is the value at that scale: a check or a
calculated field that read the value as sent would reason about a value that is never stored.

> **Normative.**
> A decimal MUST be rounded half-up to its declared scale before `checks`, `calculatedOnCreate` /
> `calculatedOnUpdate` and the pre-write validation read it, on every write path, so the value
> checked, the value calculated from and the value stored are the same value. A value already within
> its scale MUST be left untouched.
