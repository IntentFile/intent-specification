# Blank is Unicode-aware: a no-break or zero-width space alone is no value

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/163, https://github.com/eclipse-dirigible/dirigible/issues/8110
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8466

## The problem

1.11's `required` is "NOT NULL; the generated required-value validation keys on this", and
`requiredWhen` counts a value as present "when it is neither absent nor blank". Neither says which
characters make a text blank. A whitespace test that knows only the ordinary spaces lets a value made
of a no-break space (U+00A0) or a zero-width space (U+200B) through: a required `subject` of one
no-break space was trimmed to that space, passed the required check and was stored, and so was a
subject of one zero-width space. Pasted text carries these characters routinely.

## The proposed shape

No new key. The rule holds for every text a file already declares:

```yaml
- name: Ticket
  fields:
    - { name: subject, type: string, required: true }   # " " and "​" alone are refused as missing
```

## Expected behaviour

- Text is trimmed of leading and trailing whitespace before it is checked and stored, and the
  whitespace includes the no-break spaces (U+00A0, U+2007, U+202F) and the zero-width characters
  U+200B..U+200D and U+FEFF.
- A value made only of those characters is empty: a `required` text, or a `requiredWhen` value,
  made only of them is refused as missing (400), on a create and on an update, through every
  generated surface and on a task form's write-back alike.
- `"​ Edged ﻿"` is stored as `Edged`; characters inside the text are untouched.

## Edge rules

- An optional text made only of those characters is stored empty.
- The form's own client-side required rule is not the authority; the server's refusal decides.

## Prior art / workarounds

None; the invisible value was stored. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8466. An editorial statement of a decided
behaviour, tracked in the 1.11 "states precisely" checklist.

## Specification text

**Anchor:** Entities & fields > fields, after the attribute table (the `required` row), before
"label / countryLabels - a field's caption".

A text's edges are trimmed before it is checked and stored, and the whitespace a trim removes is the
whitespace a reader cannot see - the no-break and zero-width characters a paste brings along
included. A value made only of them is no value.

> **Normative.**
> Text is trimmed of leading and trailing whitespace including the no-break spaces (U+00A0, U+2007,
> U+202F) and the zero-width characters U+200B..U+200D and U+FEFF before it is checked and stored; a
> value made only of them is empty, and a `required` text made only of them MUST be refused as
> missing, on every surface and on a task form's write-back alike. The same characters make a value
> blank for `requiredWhen`.
