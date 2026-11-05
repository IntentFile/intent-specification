# A field key in any letter case - `unique: { ignoreCase: true }`

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8095
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8473

## The problem

`unique: true` compares a value exactly. Two employees whose username or e-mail address differ only
in letter case, `qalt1` and `QALT1`, are both saved, and one login then maps to two records. A file
can make the key case-insensitive today only by lower-casing what it stores (`normalize: lower`),
which rewrites a value whose spelling the reader sees, and which an identity matched against an
external claim must keep.

## The proposed shape

```yaml
fields:
  - { name: username, type: string, length: 64,  unique: { ignoreCase: true } }
  - { name: email,    type: string, length: 320, format: email, unique: { ignoreCase: true } }
```

## Expected behaviour

- A write that gives the field a value another record holds in any letter case is refused, with the
  same refusal as an exact repeat.
- The stored value keeps the case it was written in. A record may change the case of its own value.
- A record that already stores exactly the value being written is not refused, so records that held
  case variants before the key was declared stay writable until one of them changes the value.
- The rule holds on every write of the record through the generated application.
- `unique: true` means the same as `unique: {}`, and `unique: false` is unchanged.

## Edge rules

- `ignoreCase` on a field that is not a `string` or `text` is refused.
- An unknown key inside `unique: {...}` is refused.
- A field that declares no `ignoreCase` generates as before.

## Prior art / workarounds

`normalize: lower` beside `unique: true`, which makes the key case-insensitive by storing every value
lower-cased: the explicit choice where the spelling does not matter. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8473.

## Specification text

**Anchor:** Entities & fields > fields, the attribute table. The `unique` row becomes:

| `unique` | a UNIQUE constraint (e.g. a code or business key); `{ ignoreCase: true }` on a `string` / `text` field refuses a value another record holds in any letter case (see [unique: ignoreCase](#unique-ignorecase---a-key-in-any-letter-case)) |

**Anchor:** Entities & fields > fields, a new subsection before "unique - a business key over more
than one field":

#### unique: ignoreCase - a key in any letter case

`unique: true` compares a value exactly, so `QALT1` and `qalt1` are two values. A login name or an
e-mail address is one value in any letter case; declare its key so:

```yaml
- { name: username, type: string, length: 64,  unique: { ignoreCase: true } }
- { name: email,    type: string, length: 320, format: email, unique: { ignoreCase: true } }
```

A value another record holds in any letter case is refused, as an exact repeat is. The stored value
keeps the case it was written in; storing it lower-cased is `normalize: lower`, the explicit choice
where the spelling does not matter. `unique: true` is the same as `unique: {}`.

> **Normative.**
> `unique` MUST accept `true`, `false` or a mapping; a mapping MUST make the field unique, and
> `unique: true` MUST mean the same as an empty mapping. `ignoreCase: true` on a field that is not
> `string` or `text`, and an unknown key inside the mapping, MUST be reported as authoring errors.
> With `ignoreCase: true`, a conforming generator MUST refuse a write that gives the field a value
> another record holds when both are compared ignoring letter case and leading and trailing
> whitespace, with the refusal an exact repeat gets, on every write of the record through the
> generated application. It MUST store the value as written, and MUST NOT refuse a record whose
> stored value is exactly the value being written. A field without `ignoreCase` MUST generate
> byte-identical output.

## DSL index

A new row, after `entities.unique`:

| Construct | What it does |
| --- | --- |
| [`unique: { ignoreCase: true }`](#unique-ignorecase---a-key-in-any-letter-case) | a field key that refuses a value another record holds in any letter case, while the stored value keeps its case |
