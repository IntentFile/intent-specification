# 0025 — Subset relation (`subset`)

- **Status:** released in [1.6](../versions/1.6.md)
- **Issue:** _(the proposal issue, opened first)_
- **Implementation:** [eclipse-dirigible/dirigible#6878](https://github.com/eclipse-dirigible/dirigible/issues/6878) _(+ the platform PR once opened)_

## Why

A record often holds a small SUBSET of a lookup's rows — which payment methods a schedule
accepts, which channels a campaign runs on, which weekdays a rule applies to. That statement is a
reference, but it is not a to-one (several are chosen), and it earns none of what an intermediate
entity buys: there is no data on the pairing, no navigation from the lookup back to the records,
and nothing that consumes the pairs as rows. Authors today model it as a string field with a
hand-written length and pattern — the generator's storage leaking into the intent, three
attributes deep — and the generated form offers a bare text box where the user types raw numeric
keys by hand.

## What this adds

A fifth relation kind. One authored line:

```yaml
relations:
  - { name: payerTypes, kind: subset, to: PayerType, required: true }
```

`subset` is a set-valued reference to a lookup entity of the same model: the record holds a
subset of the target's rows as a single value, not as rows.

## The normative half

- **The value shape is fixed, and it is not the model's to declare.** The stored value is the
  selected target keys, comma-separated, in ascending numeric order, de-duplicated. An empty
  selection is the ABSENT value — never an empty string, never a lone delimiter. No attribute of
  the relation may author the delimiter, the ordering, or a length; a conforming parser MUST
  reject any attempt to. (The precedent is the numbering rule: the shape is not the model's to
  declare. Fixing it normatively is what makes seed data and stored values portable across
  implementations.)
- **`required` means "at least one selected."** Because an empty selection is the absent value, a
  conforming implementation maps `required: true` to a non-null constraint on the stored value.
- **Attributes.**

  | Key | Meaning |
  | --- | --- |
  | `to` | the lookup entity, declared in this model — mandatory |
  | `required` | at least one row must be selected |
  | `where` | the same single static option filter a to-one accepts, narrowing the offered rows |
  | `major` | list-column visibility, `false` keeps it off the compact list table (default true) |
  | `size` | form-control span; implementations may render the control full-width regardless |
  | `description` | free-text documentation |

- **Rejected on this kind**, each a parse error naming the kind: `composition`, `init`,
  `function`, `dependsOn`, `through`, `personal` / `partner`, calculated actions, `show`,
  `leafOnly`, and a cross-model `model:` — the stored value is the target's seed keys, which
  belong to the owner model's seeds (the same locality rule status stages follow).
- **Seeds.** A seed row may set the value by the relation's authored name; the value is the
  literal key list in the normative shape (`payerTypes: "1,3"`). A conforming parser MUST reject a
  value off that shape — seed imports typically bypass the write-path guard, so authoring time is
  the only honest place to catch it.
- **Presentation.** A conforming generator SHOULD render a multi-select over the target's rows,
  keyed by the target's primary key and labelled by its name-like field, and SHOULD normalise the
  value to the shape above on save. Wherever the stored value is read — list tables, registers,
  exports — the keys SHOULD resolve to the target's labels.
- **The boundary (the flip side of Many-to-many).** A `subset` carries no intermediate entity, no
  bridge data and no reverse navigation, and its selections are not rows — nothing that consumes
  related ROWS (per-row fan-out, reverse registers, roll-ups, reports) may consume it. A report
  dimension or filter naming a `subset` MUST be rejected: the stored value is one column, so
  grouping or comparing over it computes the wrong thing with nothing at runtime to say so. A
  uniqueness key over it MUST be rejected too — the column holds a normalized set, not an
  identity. Where any of those is wanted, model the intermediate entity.

## Notes

Deliberately deferred, so the first version stays the small construct it is: cross-model targets
(the keys belong to the owner model's seeds); seeded-NAME value lists ("name, not number" — also
the mitigation for the append-only-keys hazard a numeric list inherits); `dependsOn` on or from a
subset relation; report dimensions over the set (rejected today — an unnest is a later design); a
`max:` selections bound.

## Prior art / workarounds

A production application built on the reference implementation models exactly this as a
pattern-validated string field over a settings nomenclature — hand-authored `type`, `length` and
`pattern`, a comment noting the ids are append-only because they are embedded in the stored lists,
and a second comment noting the multi-select widget was an upstream feature still planned. The
generated back office renders it as a raw text box.

## Specification text

**Anchor:** the "Relations & multi-model" chapter, immediately after the "Many-to-many" section.

> ### Subset — a value set, not a row set
>
> The Many-to-many section holds that a real n:m is an explicit intermediate entity — a real
> entity you can read, seed and report on, which is usually what an n:m relationship needs anyway.
> *Usually* has one principled exception: the record that merely holds a SUBSET of a small
> lookup's rows. There is no data on the pairing, no navigation from the lookup back, and no
> consumer of the pairs as rows — the intermediate entity would be pure overhead, and the author
> would be writing a real entity to express a checkbox group.
>
> `subset` says that directly:
>
> ```yaml
> relations:
>   - { name: payerTypes, kind: subset, to: PayerType, required: true }
> ```
>
> The record stores the selected target keys as one value — comma-separated, in ascending numeric
> order, de-duplicated; an empty selection is the absent value, never an empty string. That shape
> is normative and not authorable: no attribute may set the delimiter, the ordering or a length,
> and `required` means at least one selected (the stored value is non-null). A seed row sets the
> value by the relation's authored name in the same shape (`payerTypes: "1,3"`).
>
> The target must be an entity of this model; `where` narrows the offered rows exactly as it does
> on a to-one; `major` and `size` keep their meanings. Everything that describes a to-one foreign
> key or a row set is rejected at parse: `composition`, `init`, `function`, `dependsOn`,
> `through`, `personal` / `partner`, calculated actions, `show`, `leafOnly`, and a cross-model
> `model:`.
>
> A conforming generator SHOULD render a multi-select over the target's rows and SHOULD resolve
> the stored keys to the target's labels wherever the value is read. The moment the pairing
> carries data, needs reverse navigation, or must be consumed as rows — per-row fan-out, reverse
> registers, roll-ups, reports — it has outgrown a value: model the intermediate entity. A report
> dimension or filter over a `subset`, and a uniqueness key spanning it, are rejected for the
> same reason.

**Anchor:** Appendix A: DSL index.

| Construct | What it gives you |
| --- | --- |
| [`subset`](#subset--a-value-set-not-a-row-set) | a set-valued reference to a same-model lookup: the record stores the selected keys as one comma-separated, ascending, de-duplicated value; empty = absent |

_(and the existing `relations[].kind` row's allowed values gain `subset`)_
