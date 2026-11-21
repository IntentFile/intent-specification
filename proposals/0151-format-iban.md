# An IBAN is checked, not only shaped - `format: iban` on a field

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8302
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8706

## The problem

An IBAN field can only be validated by `pattern:`, and a regular expression checks the shape:

```yaml
- { name: iban, type: string, length: 34, pattern: '^[A-Z]{2}[0-9]{2}[A-Z0-9]{11,30}$' }
```

The ISO 13616 check digits are never verified. `BG00BNBG96611020345678` has the right shape. Its
check digits `00` can never be valid, and the number fails ISO 7064 MOD 97-10. It is still stored,
and later used as a payment destination. The same applies to every IBAN a model carries: an own bank
account, a customer's or a supplier's account, an employee's salary account.

A checksum is an algorithm, and an algorithm is implementation code. A hand-written hook that
computes a field makes that field read-only on the generated form, so a value the user types cannot
be checked this way. Validating an account number is common enough to be a named format.

## The proposed shape

```yaml
- name: Employee
  fields:
    - { name: id,   type: integer, primaryKey: true, generated: true }
    - { name: name, type: string, length: 120, required: true }
    - { name: iban, type: string, length: 34, format: iban, normalize: [strip: " ", upper] }
```

## Expected behaviour

- `format: iban` is a second named input format beside `format: email`, on a `string` field.
- A value is accepted when it has the IBAN shape in its electronic form (two letters, two digits,
  11 to 30 letters and digits) **and** its check digits hold. When the first four characters are
  moved to the end and every letter is read as 10 to 35, the number leaves a remainder of 1 when
  divided by 97.
- Any other value is refused on a create and on an update. The refusal is translated like the other
  refusals of one field's value ([0145](0145-field-refusals-name-their-field.md)), names the field
  and carries its translation key, so a form marks the field.
- The value is checked as stored: no spaces, upper case. To accept the printed form
  (`BG80 BNBG 9661 1020 3456 78`), declare `normalize: [strip: " ", upper]`. Normalization runs
  before the check.
- The form input carries the shape, so a value off it is reported before the record is submitted.
  The check digits are verified by the server.
- A blank value of an optional field is not checked, as for `pattern`.
- A model without `format: iban` generates exactly as before.

## Edge rules

- `format` and `pattern` on the same field are an authoring error, as for `format: email`.
- `format: iban` on a field that is not a `string` is an authoring error.

## Prior art / workarounds

- A `pattern:` checks the shape only.
- A computed-field hook makes the field read-only on the form.
- A general field-level validator hook (`validate: <Class>`) would also cover national identifiers
  and other checksums. It is not proposed here.

Reference implementation: https://github.com/eclipse-dirigible/dirigible/pull/8706.

## Specification text

**Anchor:** Data model > fields > field attributes: the `format` row.

Table row (replaces the `format` row):

| `format` | a **named input format** over `pattern`: `email` (the canonical address shape and an e-mail input) or `iban` (the IBAN shape and its ISO 7064 MOD 97-10 check digits); enforced server-side and refused like an authored `pattern`; mutually exclusive with `pattern` |

> **Normative.**
> A conforming generator MUST refuse, on a create and on an update, a non-blank value of a field
> declaring `format: iban` unless it matches `^[A-Z]{2}[0-9]{2}[A-Z0-9]{11,30}$` and the number
> formed by moving its first four characters to the end and replacing every letter by its position
> plus 9 (`A` = 10 ... `Z` = 35) leaves a remainder of 1 when divided by 97. The value MUST be checked
> after the field's `normalize` transforms. The refusal MUST name the field and MUST be resolved for
> the reader's language. `format: iban` on a field that is not a `string`, or beside `pattern`, is an
> authoring error reported at parse time. A model declaring no `format: iban` MUST generate as before.

## DSL index

| Construct | What it does |
| --- | --- |
| [`format`](#fields) | a named input format over `pattern`: `email`, or `iban` (shape and check digits) |
