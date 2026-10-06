# A header value required when any line matches - `requiredWhen` with `whenAnyItem`

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/103, https://github.com/eclipse-dirigible/dirigible/issues/7560
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7630

## The problem

A tax law says: a zero-rated or reverse-charged **line** makes the **document's** legal ground a
requisite. The condition is on the lines; the required value is on the header.

`requiredWhen` ([0036](0036-a-value-required-only-under-a-condition.md)) guards on the record's own
properties or one to-one hop - it cannot look down into the items. `itemsCompare` (0062) reads every
line but is a warning and compares a line against a literal. A roll-up flag would need an ordering
operator the `when` grammar does not have. So an invoice with a 0% line is issued without the
requisite, and the omission is found by the auditor.

## The proposed shape

```yaml
- name: SalesInvoice
  checks:
    - { kind: requiredWhen, field: vatGround, whenAnyItem: "vatRate == 0", status: ISSUED,
        message: "A zero-rated line needs its legal ground" }
```

## Expected behaviour

`whenAnyItem:` is an alternative to `when:` on a `requiredWhen` of a document entity. It has the
same `<Property> ==|!= <literal>` grammar, read off **each line** of the document's items child; the
header's `field` is required when at least one line satisfies it. Everything else about the check is
0036's: the presence rule, the `status:` gate, the message, the refusal reaching whoever acted.

## Edge rules

- Exactly one of `when:` and `whenAnyItem:` is given; both or neither is an authoring error.
- The entity must have a composition items child; the property named must be the child's own field
  or to-one key, of a type an equality is exact on (0036's list).
- A document with no lines satisfies nothing: the value is not required.
- With a `status:` gate the check runs when the document is persisted carrying that status, after
  its lines are stored - which is where a lines-dependent rule can be judged at all. Without a gate
  it runs on every write of the header **and** on every write of a line, since a line can make the
  condition true.

## Prior art / workarounds

A hand-written check in the document's custom code. Reference implementation: https://github.com/eclipse-dirigible/dirigible/pull/7630.

## Specification text

**Anchor:** Entities & fields > checks > kind: requiredWhen - a value required only under a
condition, appended to the bullet list and the normative block.

- `whenAnyItem` - instead of `when`: the same comparison, read off each line of the document's
  items child. The value is required when at least one line satisfies it:

  ```yaml
  - { kind: requiredWhen, field: vatGround, whenAnyItem: "vatRate == 0", status: ISSUED,
      message: "A zero-rated line needs its legal ground" }
  ```

> **Normative.**
> A `requiredWhen` MUST carry exactly one of `when` and `whenAnyItem`. With `whenAnyItem` the
> condition MUST be evaluated against every row of the entity's composition items child, holding when
> any row satisfies it and not holding for a document without lines; the named property MUST be the
> child's own field or to-one key and is subject to the same type rules as `when`. With a `status`
> gate the check MUST run when the document is persisted carrying that status, after its lines are
> stored; without one it MUST run on every user write of the document and of its lines. An entity
> without a composition items child MUST report `whenAnyItem` as an authoring error.

## DSL index

| Construct | What it does |
| --- | --- |
| [`requiredWhen: whenAnyItem`](#kind-requiredwhen--a-value-required-only-under-a-condition) | a header value required when any line of the document satisfies a comparison |
