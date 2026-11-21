# What a value off its pattern is refused with - `patternMessage:` on a field

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7749
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8654

## The problem

A field's `pattern:` is an input-format regular expression, enforced on the form and on the server.
A value that does not match it is refused with a generic sentence that quotes the regex:

```
The value of 'Registration number' does not match the required pattern '^([0-9]{9}|[0-9]{10}|[0-9]{13})$'
```

The sentence is translated and names the field ([0145](0145-field-refusals-name-their-field.md)), but it
still describes the regex, not the rule. The author cannot say what the rule means - "a company
registration number has 9, 10 or 13 digits" - and a reader who does not read regular expressions
cannot tell what to type. Every `checks:` rule carries a translatable `message:`; a field pattern
carries none.

## The proposed shape

```yaml
- name: Customer
  fields:
    - { name: id,   type: integer, primaryKey: true, generated: true }
    - { name: name, type: string, length: 120, required: true }
    - name: registrationNumber
      type: string
      length: 13
      label: Registration number
      pattern: '^([0-9]{9}|[0-9]{10}|[0-9]{13})$'
      patternMessage: "The {label} has 9, 10 or 13 digits"
```

## Expected behaviour

- A value that does not match the field's `pattern` is refused with the declared message instead of
  the generic sentence, on the server and on the form. On the form the message is the hint the input
  shows for a value off its pattern.
- The message is written in the default language, as a `checks:` message is, and seeds the
  default-language catalogue, so a language catalogue translates it. It is resolved for the reader's
  language; an entry a language does not translate falls back to the default-language text.
- `{label}` in the message is the field's caption in the reader's language, `{pattern}` the regular
  expression.
- The refusal names the field it is about and carries its translation key and its parameters, as the
  other refusals of one field's value do (0145), so a form marks the field.
- A field declaring `pattern` without `patternMessage` is refused with the generic sentence, as
  before. A model without `patternMessage:` generates exactly as before.

## Edge rules

- `patternMessage:` without `pattern:` on the same field is an authoring error: there is no pattern
  for the message to explain. A `format:` preset (`email`) keeps its own message.
- A blank `patternMessage:` is an authoring error (remove it to keep the generic sentence).
- When to refuse does not change: a blank value of an optional field is still not checked against the
  pattern.

## Prior art / workarounds

The generic sentence with the regex, or a `checks:` rule that duplicates the pattern only to carry a
message - which no check kind can do, since none takes a regular expression. Reference
implementation: https://github.com/eclipse-dirigible/dirigible/pull/8654.

## Specification text

**Anchor:** Data model > fields > field attributes: a table row after `pattern`.

Table row:

| `patternMessage` | what a value off the field's `pattern` is refused with, instead of the generic sentence quoting the regex; translated like a check message, `{label}` and `{pattern}` interpolated |

> **Normative.**
> A conforming generator MUST refuse a value that does not match the `pattern` of a field declaring
> `patternMessage: <text>` with that text instead of its generic pattern refusal, on the server and on
> the form input that carries the pattern. The text MUST seed the default-language catalogue and MUST
> be resolved for the reader's language, `{label}` as the field's caption in that language and
> `{pattern}` as the regular expression, falling back to the default-language text. The refusal MUST
> name the field. `patternMessage:` without `pattern:` on the same field, and a blank
> `patternMessage:`, are authoring errors reported at parse time. A model declaring no
> `patternMessage:` MUST generate as before.

## DSL index

| Construct | What it does |
| --- | --- |
| [`patternMessage`](#fields) | what a value off a field's `pattern` is refused with, translated |
