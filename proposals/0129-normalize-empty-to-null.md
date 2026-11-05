# `normalize: emptyToNull` stores a blank value as no value, so an optional unique field holds any number of them

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8492
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8527

## The problem

An optional `string` field with `unique: true` stores a blank value as an empty string, not as no
value. A uniqueness key lets any number of rows hold no value, but it compares the empty string like
any other value. So the first record saved with a blank takes the empty string, and every later
record saved with a blank is refused as a duplicate, although none of them has a value.

`normalize` (proposal 0087) offers `trim`, `strip` and a case fold. `trim` turns `"   "` into `""`,
never into no value. A `required` field refuses a blank, and an optional one stores it as it is.

A business suite met this on a company's VAT and registration numbers: `unique`,
`normalize: [strip: " ", upper]`, `pattern`. Creating a company with an empty VAT number twice saved
the first and refused the second.

## The proposed shape

```yaml
fields:
  - { name: vatNumber, type: string, length: 20, unique: true, normalize: [trim, emptyToNull], pattern: '^[A-Z]{2}[0-9]{8,10}$' }
```

## Expected behaviour

- `emptyToNull` is one more `normalize` transform. It runs last, after trim, strip and the case
  fold, wherever it is listed.
- A blank result is stored as no value (null). Blank means empty, or made only of whitespace,
  including the Unicode space separators (a no-break space) and the zero-width characters.
- It is applied on every write of the record, like every `normalize` transform. The uniqueness key,
  `pattern` (which does not judge a missing value) and `checks` such as `requiredWhen` all read no
  value.
- Two records created with `""` and two with `"   "` are all saved, without a value. Two equal real
  values still collide. A real value that does not match `pattern` is still refused.

## Edge rules

Refused when the intent is read:

- `emptyToNull` on a `required` field. A required field refuses a blank value, so the transform
  could never store the no value it promises.
- `emptyToNull` with an argument.

A field without `emptyToNull` generates byte-identically.

Some data stores let a uniqueness key admit only one row without a value. On such a store, the
second blank still collides. That follows from the store's semantics, not from this format.

## Prior art / workarounds

A `forbidWhen` with `verbs: [create, update]` and `when: ["vatNumber == ''"]` refuses the empty
string. That stops the collision, but it refuses every record without a value instead of saving it.

## Specification text

**Anchor:** Entities & fields > fields. Amends the `normalize` row of the attribute table and appends
to the subsection "normalize - the stored form" (both from proposal 0087).

The row becomes:

| `normalize` | the transforms that bring a `string` / `text` value to its stored form before the length check, `pattern` and `checks` read it - `trim`, `strip: "<characters>"`, `upper`, `lower`, `emptyToNull` (see [normalize](#normalize---the-stored-form)) |

Appended to the subsection, after its example:

`emptyToNull` stores a blank result as no value. An optional `unique` field needs it. A uniqueness
key lets any number of rows hold no value, but it compares an empty string like any other value, so
without `emptyToNull` the second record saved without a value is refused as a duplicate:

```yaml
- { name: vatNumber, type: string, length: 20, unique: true, normalize: [trim, emptyToNull] }
```

Appended to the subsection's normative block:

> **Normative.**
> `emptyToNull` MUST run after every other `normalize` transform, whatever position it is listed
> in. It MUST store a blank result as no value (null). A blank result is empty, or made only of
> whitespace, Unicode space separators or zero-width characters. The uniqueness key, `pattern` and
> `checks` MUST read that no value. A generator MUST reject `emptyToNull` on a `required` field, and
> `emptyToNull` with an argument.

## DSL index

The `normalize` row is amended:

| Construct | What it does |
| --- | --- |
| [`normalize`](#normalize---the-stored-form) | a `string` / `text` field's stored form - trim, strip characters, fold case, a blank stored as no value - applied on every write before the length check, `pattern` and `checks` read the value |
