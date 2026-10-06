# An authored `id` keys a check's message in the translation catalog

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/109, https://github.com/eclipse-dirigible/dirigible/issues/7611
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7621

## The problem

A check's `message:` is one string, in the module's default language, and it reaches the reader in
that language whatever language they read the interface in. Once it is translated through the
catalog, the catalog needs a key, and the only key a check has is its position:
`Invoice_itemsCompare_0`. Insert a check above it and every key below moves; the translator's work
attaches to the wrong rule or is lost.

The tempting fix - a per-language map in the file, `message: { en: ..., bg: ... }` - makes every
check an N-language block of interface copy inside the model, puts a translator's edit behind a
regeneration, and leaves one key with two sources of truth. The format already decided this for
`label:` (0031): the model declares structure, the catalogs carry the languages.

## The proposed shape

```yaml
- name: Invoice
  checks:
    - { id: zeroLines, kind: itemsCompare, field: price, op: gt, value: 0,
        message: "{count} line(s) at price zero" }
```

## Expected behaviour

`id:` is an optional stable name for a check. Its catalog key becomes `<Entity>_<id>` instead of the
positional one, so the translation survives any reordering. The authored `message` seeds the
default-language catalog entry, and the refusal or warning reaches the reader in their language where
the catalog has it. The same rule covers a picker rule's message and a business key's message.

## Edge rules

- An `id` repeated within one entity is an authoring error.
- A per-language map as `message` is refused with a message naming the catalog path.
- Placeholders (`{count}`, `{match}`, `{capacity}`) are resolved after the translation is chosen.
- A check without `id` keeps its positional key; adding an `id` later moves the entry.

## Prior art / workarounds

Editing the generated catalog by key, redone after every reorder. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/7621.

## Specification text

**Anchor:** Entities & fields > checks - declarative validations, appended to the introduction.

A check may carry an `id`, a stable name that keys its message in the translation catalog:

```yaml
- { id: zeroLines, kind: itemsCompare, field: price, op: gt, value: 0, message: "{count} line(s) at price zero" }
```

The message stays one string in the module's default language; the catalogs carry the other
languages, as for a `label:`.

> **Normative.**
> A conforming generator MUST seed the default-language catalog with every authored check, picker
> rule and business-key message, keyed `<Entity>_<id>` where an `id` is declared and by position
> otherwise, and MUST deliver a refusal or warning in the reader's language where the catalog has
> an entry, resolving placeholders after the translation is chosen. An `id` repeated within an
> entity, and a `message` that is a mapping rather than a string, MUST be reported as authoring
> errors, the latter naming the catalog as the place for other languages.

## DSL index

| Construct | What it does |
| --- | --- |
| [`checks[].id`](#checks--declarative-validations) | a stable name that keys the check's message in the translation catalog |
