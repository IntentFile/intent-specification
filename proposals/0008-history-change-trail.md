# `history` - the shadow change trail


- **Status:** released in [1.3](../versions/1.3.md)
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/6715
- **Discussion:** https://github.com/IntentFile/intent-specification/pull/17

`audit: true` records only the **last** writer and time, in four columns of the row itself. A domain
that has to answer *what changed, from what to what, by whom, when* — for every write, for years —
had no construct for it; the 1.1 "Planned" list named shadow audit-history entities, and this is
that construct.

## What it adds

A `### history — the change trail` section in `versions/1.2.md`, plus the entity-attribute row and
the *Appendix A: DSL index* row; the "Planned" entry loses the item.

```yaml
- name: Contract
  audit: true
  history: true
```

The entity gains a **shadow history table** — a sibling of its own table, like the multilingual
language table — carrying one entry per property whose value actually changed on a write: the
property, its old value, its new value, who wrote it, when, and whether the write came from a **user**
or from the **system**. A create is `null -> value` and a delete `value -> null`, so the trail alone
reconstructs the row at any point in its life; the record's own form shows it as a read-only History
panel.

## The normative rules, and why each one is there

- **Append-only by construction** — no create, update or delete path may be emitted for the shadow
  table, not a service, not an endpoint, not a UI affordance. Append-only enforced by policy is not
  append-only.
- **Every write path appends**, including the targeted single- and multi-column writes the system
  uses. A path that writes silently is worse than no trail, because the trail then reads as complete.
- **The source is recorded.** Once a total the application recomputed and an amount a person typed
  sit in the same column, nothing downstream can tell them apart.
- **Representation-only differences are not changes** — a decimal of another scale, a translated
  overlay of a stored value. Otherwise the trail fills with edits nobody made.
- **The primary key and the audit columns are not tracked** — they restate what the entry carries.
- **A scoped surface that hides `sensitive:` fields is not handed a trail it cannot filter** — either
  the trail excludes those properties or the surface exposes none. Leaking a hidden field's old and
  new values defeats the scoping exactly.
- **Rows written outside the generated data-access layer have no history**, and a conforming tool
  says so rather than implying completeness.

## Reference implementation

eclipse-dirigible/dirigible#6734 — schema-level shadow table, appends on every repository write path
with the user/system attribution, `GET /{id}/history`, the History panel, and an end-to-end test
covering both the user and the system side.

Left open for maintainer review.

## Specification text

The prose below is what a release folds into the next version document, at the anchors
given. It was written against `versions/1.2.md` and is carried here unchanged.

**Anchor:** Entities & fields > fields > Entity-level attributes

| `history: true` | records every write as field-level deltas in a shadow history table (see [history](#history--the-change-trail)) |

**Anchor:** stamped at a modeled issue step (a placeholder holds the field until then): > locksWithMaster — a child collection that outlives its master's lock


### history — the change trail

`audit: true` records only the **last** writer and time, in four columns of the row itself. Where a
domain has to answer *what changed, from what to what, by whom, when* — for every write, for years —
declare a history:

```yaml
- name: Contract
  audit: true
  history: true                  # every write is recorded as field-level deltas
  fields:
    - { name: id,     type: integer, primaryKey: true }
    - { name: amount, type: decimal }
```

The entity gains a **shadow history table** — a sibling of its own table, like the
[multilingual](#multilingual-data) language table — carrying one entry per property whose value
actually changed on a write: the property, its old value, its new value, who wrote it, when, and
whether the write came from a **user** or from the **system** (a roll-up total, a workflow
write-back, a recomputed document total). A create is recorded as `null -> value` and a delete as
`value -> null`, so the trail alone reconstructs the row at any point in its life. The record's own
form shows it as a read-only **History** panel.

The source matters as much as the delta. Once a total the application recomputed and an amount a
person typed sit in the same column, nothing downstream can tell them apart — and "who changed this"
is the first question asked of a trail.

> **Normative.**
> The shadow table is **append-only by construction**: a generator MUST NOT emit any create, update
> or delete path to it — not a service, not an endpoint, not a UI affordance. Append-only enforced by
> policy is not append-only.
> Every write path the generated data-access layer offers MUST append, including the targeted
> single-column and multi-column writes the system uses; a path that writes silently is worse than
> no trail, because the trail then reads as complete.
> An entry MUST record whether the write was a user write or a system write.
> Only properties whose value actually changed are recorded. Values that differ solely in
> representation (a decimal of a different scale, a translated overlay of a stored value) are NOT
> changes, and a generator MUST NOT record them as such.
> The primary key and the audit columns are NOT tracked — the key never changes and the audit columns
> restate what the entry already carries.
> A [scoped surface](#personal-and-partner-surfaces) that hides `sensitive:` fields MUST NOT be given
> a history it cannot filter: either the trail it exposes excludes those properties, or it exposes
> none. Leaking a hidden field's old and new values defeats the scoping exactly.
> Rows written outside the generated data-access layer — [seeds](#seeds), direct database writes —
> have no history, and a conforming tool documents that rather than implying completeness.

**Anchor:** Appendix A: DSL index

| [`history`](#history--the-change-trail) | a shadow, append-only trail of every write: property, old and new value, who, when, user or system |

**Anchor:** Appendix A: DSL index > Planned — recognised but not yet implemented

- Event-driven **document** generation (produce a whole document, rather than the mapped rows of [`posts`](#posts--derived-rows-on-an-event), on an event) and a declarative state machine.
