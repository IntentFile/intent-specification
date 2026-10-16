# A roll-up counts the rows it is told to, and its ceiling is enforced at a status - `where`, `guardAt`

- **Status:** released in [1.11](../versions/1.11.md)
- **Issue:** https://github.com/IntentFile/intent-specification/issues/102, https://github.com/eclipse-dirigible/dirigible/issues/7542
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7600

## The problem

Two things are wrong with a capacity roll-up as specified, and they meet on the same document.

A roll-up counts **every** child. A cancelled call-off goes on consuming its contract's capacity for
ever, a voided allocation goes on paying its invoice; the fund stays exhausted and the replacement
document can never be issued, with every write succeeding. There is no way to say which rows count.

And the overdraw guard runs on every write of the child. That is right for a row that carries its own
typed amount - an allocation - and useless when the child's amount is a **document total**, recomputed
from its lines after the header is written: the guard sees the zero the header was created with, the
lines are added, the total grows past the ceiling, and nothing ever checks it. The ceiling is
unenforced in exactly the case it was declared for.

## The proposed shape

```yaml
rollups:
  - name: contractCommitted
    entity: CallOff
    via: Contract
    field: committed
    op: sum
    of: total
    capacity: ceiling
    where:
      - { field: Status, op: ne, value: CANCELLED }   # eq / ne only; a list is their AND
    guardAt: ISSUED
    message: "Only {remaining} left of {capacity}; this call-off asks {requested}."
```

## Expected behaviour

`where:` narrows the rows the roll-up reduces: one or more `{ field, op, value }` terms over the
child's own properties, `op` being `eq` or `ne`, a status named by its seeded name, a list being
their AND. It is the same shape `schedules[].where` takes. A child that leaves the set - cancelled,
re-pointed, deleted - recomputes the parent exactly as a delete does, and one that enters it as a
create does.

`guardAt: <status>` moves the capacity check from every write of the child to the moment the child
is persisted **carrying** that status, after its own derived values are computed. Without it the guard
runs on every write, as before. `message:` is the refusal's text; `{capacity}`, `{sum}`,
`{remaining}` and `{requested}` are resolved into it.

## Edge rules

- `where:` without `op: sum` or a count is fine; it narrows a count too.
- `guardAt` requires `capacity` and a `function: EntityStatus` relation on the child; it names a
  status of the child's own nomenclature.
- A `guardAt` child that is already past the guarded status is not re-checked on later writes; a
  move back into it is.
- The guard reads the sum of the rows that satisfy `where:`; a row the filter excludes neither
  consumes capacity nor is refused for exceeding it.
- `where:` on a cross-model child (0028) reads the child's own properties, which are local.

## Prior art / workarounds

A status-aware hand-written recompute, which disagrees with the generated one about which rows count.
Reference implementation: https://github.com/eclipse-dirigible/dirigible/pull/7600.

## Specification text

**Anchor:** Declarative glue > rollups - denormalised parent totals, after the normative paragraph on
re-parenting.

A roll-up may say which children count, and when its ceiling is checked:

```yaml
rollups:
  - { name: contractCommitted, entity: CallOff, via: Contract, field: committed, op: sum, of: total,
      capacity: ceiling,
      where: [ { field: Status, op: ne, value: CANCELLED } ],
      guardAt: ISSUED,
      message: "Only {remaining} left of {capacity}; this call-off asks {requested}." }
```

`where:` is a list of `{ field, op, value }` terms over the child's own properties, `eq` or `ne`,
their AND; only the rows that satisfy it are reduced. `guardAt:` names the child's status at which
the `capacity` check runs - for a child whose amount is a document total that only exists once its
lines are in - instead of on every write. `message:` is the refusal's text, with `{capacity}`,
`{sum}`, `{remaining}` and `{requested}` resolved.

> **Normative.**
> A roll-up declaring `where:` MUST reduce only the children that satisfy every term, MUST recompute
> the parent when a child enters or leaves the set as it does on a create or a delete, and MUST
> count a row the filter excludes against neither the total nor the capacity. `op` MUST be `eq` or
> `ne`; a status value MAY be named by its seeded name. A roll-up declaring `guardAt:` MUST evaluate
> its capacity check when the child is persisted carrying that status, after the child's own derived
> values are computed, and MUST NOT evaluate it on other writes; without `guardAt:` the check MUST
> run on every write of the child as before. A refusal MUST carry `message` with its placeholders
> resolved, or the implementation's default text when none is authored. `guardAt:` without
> `capacity`, `guardAt:` on a child without a `function: EntityStatus` relation, a `where:` term
> naming a property the child does not carry, and an `op` other than `eq` / `ne` MUST be
> reported as authoring errors.

## DSL index

| Construct | What it does |
| --- | --- |
| [`rollups[].where`](#rollups--denormalised-parent-totals) | which child rows a roll-up reduces - `{ field, op, value }` terms, `eq` / `ne`, their AND |
| [`rollups[].guardAt` / `message`](#rollups--denormalised-parent-totals) | the child status at which a capacity ceiling is checked, and the refusal's text |
