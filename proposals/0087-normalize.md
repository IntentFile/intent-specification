# `normalize` brings a string field to its stored form before `pattern` and `checks` read it

- **Status:** released in [1.13](../versions/1.13.md)
- **Issue:** https://github.com/IntentFile/intent-specification/issues/148, https://github.com/eclipse-dirigible/dirigible/issues/7726
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7730

## The problem

`pattern` validates the raw input. A pattern strict enough to guarantee the stored format
(`^\+[1-9][0-9]{6,14}$`) refuses the way people paste a phone number - `+359 898 123 456`,
`+359-898-123-456` - and a pattern loose enough to accept them stores the separators, so two
spellings of one IBAN are two values. 1.11 describes `pattern` as "an input-format regular
expression the value must match" and has no declarative step between the input and its
validation; the only way to store one form is a hand-written calculated-field call-out.

## The proposed shape

```yaml
fields:
  - { name: phone, type: string, length: 20, normalize: [strip: " -()"], pattern: '^\+[1-9][0-9]{6,14}$' }
  - { name: iban,  type: string, length: 34, normalize: [strip: " ", upper] }
  - { name: code,  type: string, length: 10, normalize: lower }       # one bare transform
```

## Expected behaviour

- `normalize` on a `string` / `text` field is one transform or a list of them: `trim` (leading and
  trailing whitespace), `strip: "<characters>"` (every occurrence of each listed character, wherever
  it stands), `upper` and `lower` (the case fold).
- The transforms run in a fixed order - trim, then strip, then the case fold - whatever order they
  are listed in.
- The step is applied on every write of the record - an endpoint write, a process step's
  write-back, a system write - so one value has one stored form.
- The length check, `pattern` and `checks` read the value **as it will be stored**:
  `+359 898-123 (456)` is accepted and stored as `+359898123456`, ` bg80 bnbg 9661 1020 3456 78 ` as
  `BG80BNBG96611020345678`, and `+359 898 12a` is still refused by the pattern (400).
- A form applies the same step when the input loses focus, so what the user sees is what is stored.

## Edge rules

Refused when the intent is read:

- `normalize` on a field that is not `string` / `text`;
- an empty list;
- a transform listed twice;
- both `upper` and `lower`;
- `strip` without characters;
- an argument on a bare transform (`trim`, `upper`, `lower`);
- an unknown transform.

A field without `normalize` generates byte-identically.

## Prior art / workarounds

A loose `pattern` that stores the separators, or a hand-written calculated-field call-out that
rewrites the value. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/7730.

## Specification text

**Anchor:** Entities & fields > fields. A row in the attribute table, after `format`; and a
subsection after the table, before "label / countryLabels - a field's caption".

| `normalize` | the transforms that bring a `string` / `text` value to its stored form before the length check, `pattern` and `checks` read it - `trim`, `strip: "<characters>"`, `upper`, `lower` (see [normalize](#normalize---the-stored-form)) |

#### normalize - the stored form

A `pattern` judges the value it is given. Where people type one value several ways - a phone number
with spaces or dashes, an IBAN in lower case - declare the form it is stored in, and the pattern
judges that:

```yaml
- { name: phone, type: string, length: 20, normalize: [strip: " -()"], pattern: '^\+[1-9][0-9]{6,14}$' }
- { name: iban,  type: string, length: 34, normalize: [strip: " ", upper] }
- { name: code,  type: string, length: 10, normalize: lower }
```

`normalize` is one transform or a list of them. `trim` removes leading and trailing whitespace;
`strip` removes every occurrence of each character it lists; `upper` and `lower` fold the case. They
apply in that order - trim, strip, the case fold - whatever order they are written in. A form applies
the same step when the input loses focus, so the user sees the stored form before saving.

> **Normative.**
> `normalize` MUST be accepted on a `string` or `text` field only. A conforming generator MUST apply
> its transforms in the fixed order trim, strip, case fold, on every write of the record through the
> generated application, so that every writer stores the same form, and MUST apply them before the
> length check, `pattern` and `checks` read the value: each of them judges the value as it will be
> stored. A value the transforms leave non-matching is still refused by `pattern`. A generated form
> MUST apply the same transforms when the input loses focus. A generator MUST reject as an authoring
> error: `normalize` on any other type; an empty list; a transform listed twice; `upper` together
> with `lower`; a `strip` with no characters; an argument on `trim`, `upper` or `lower`; and an
> unknown transform. A field declaring no `normalize` MUST generate byte-identical output.

## DSL index

| Construct | What it does |
| --- | --- |
| [`normalize`](#normalize---the-stored-form) | a `string` / `text` field's stored form - trim, strip characters, fold case - applied on every write before the length check, `pattern` and `checks` read the value |
