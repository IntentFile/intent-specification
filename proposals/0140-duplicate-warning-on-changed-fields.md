# A duplicate warning asks an update only when it changes one of the checked fields

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8568
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8572

## The problem

A `duplicate` check warns before a second record carries the same `fields`, and the person saving
confirms it when the repeat is intended - two companies may share a registered name. 1.12 says
what the check compares and that it excludes the row being updated from its own search, but not
when an update is asked at all. So once the second record exists, the twin is still there on every
later save of either record, and every save asks the same question again - changing only the
price of a product whose name was confirmed as a duplicate when it was created is answered with
the duplicate warning.

The hard counterpart already draws the line: a field key in any letter case does not refuse again
a value the row already stores.

## The proposed shape

No new key. The rule holds for every `duplicate` a file already declares:

```yaml
- name: Product
  checks:
    - { kind: duplicate, fields: [name], message: "A product with this name already exists: {match}" }
```

## Expected behaviour

- A create is asked whenever another record carries the same `fields`, as today.
- An update is asked only when it changes at least one of the check's `fields` from the stored
  row, and then exactly as a create is.
- "Changes" is decided the way the check compares: a `string` / `text` field that differs from the
  stored value only in letter case or leading and trailing whitespace is unchanged; every other
  field is compared by value.

## Edge rules

- An update that changes a checked field into another record's value is asked, the confirmed twin
  included.
- Only `duplicate` is concerned. The other soft checks judge the record as it is written, not
  against other rows, and keep being asked on every write that breaks them.

## Prior art / workarounds

The confirmation was asked again on every save; the caller had to confirm the same code each time.
Reference implementation: https://github.com/eclipse-dirigible/dirigible/pull/8572.

## Specification text

**Anchor:** Entities > checks > Warn and confirm - `severity: warn`, `duplicate`, `itemsCompare`.
The prose sentence is appended to the paragraph that describes `duplicate`; the normative sentence
is appended to the `duplicate` sentence of the normative block, after "MUST exclude the row being
updated from its own search".

An update is asked about a `duplicate` only when it changes one of its `fields`, compared the same
way, so a duplicate confirmed once is not asked again on every later save of the record.

> **Normative.**
> On an update, a conforming generator MUST evaluate a `duplicate` only when at least one of its
> `fields` differs from the stored row - a `string` / `text` member compared ignoring case and
> leading and trailing whitespace, every other member by value - and MUST evaluate it on every
> create.
