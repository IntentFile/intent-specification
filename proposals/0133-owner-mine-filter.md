# A Mine filter on the regular list - `owner: true` on a to-one relation

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8499
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8525

## The problem

The only construct that knows the current user is `personal: true`, and it adds a second, personal
surface: right for employee self-service (one's own leave, timesheets, payslips), wrong for a
business register. A sales rep wants "my quotations" on the regular list of quotations, beside the
colleagues' rows that the rep's role already lets them read. A file that declared `personal:` only
to get that view also opened a personal surface through which every employee writes their own
records.

## The proposed shape

```yaml
entities:
  - name: Employee
    identity: email
  - name: Quotation
    relations:
      - { name: SalesRep, kind: manyToOne, to: Employee, owner: true }
```

## Expected behaviour

- The regular list of `Quotation` offers **All** and **Mine**; All is the default, so nothing
  changes for a reader who never picks Mine.
- Mine shows only the rows whose `SalesRep` is the caller's identity record, resolved as the
  personal surface resolves it. The list, its count and its search are narrowed alike, so the
  page, the order and the column filters work on the caller's rows.
- A caller whom no identity record maps owns no row: Mine is empty, never every row.
- The regular surface's roles apply first and unchanged. Mine only narrows what they already let
  the caller read.
- No surface, page or write path is added. The relation stays an ordinary editable relation, and
  nothing is inherited through it.

## Edge rules

- `owner: true` is accepted on a `manyToOne` or `oneToOne` relation whose target declares
  `identity`, at most once per entity.
- Refused on a `subset`, a `manyToMany`, a second owner relation of the same entity, and a target
  without `identity` (for a target owned by another model, when the application is generated).
- A relation may carry both `owner: true` and `personal: true`.
- An entity without an owner relation generates as before.

## Prior art / workarounds

`personal: true`, which gives the owner a view of their rows at the cost of a whole personal
surface with its own write path. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8525.

## Specification text

**Anchor:** Scoped surfaces & roles > Personal and partner surfaces, a new subsection after the
section's normative block (before "A user task can also be routed to the record owner's inbox").

#### owner - a Mine filter on the regular list

A to-one relation to an `identity`-declaring entity MAY declare `owner: true`. The entity's regular
list then offers **All** and **Mine**, Mine showing the rows whose relation names the caller's
identity record - a sales rep's own quotations, an agent's own tickets - beside the rows the rep's
role already lets them read:

```yaml
- { name: SalesRep, kind: manyToOne, to: Employee, owner: true }
```

Unlike `personal:`, `owner:` adds no surface: it is a filter on the regular one, not a scope. The
regular surface's roles decide what the caller may read, and Mine narrows that. A relation may carry
both keys, when the owner also needs the personal surface.

> **Normative.**
> `owner: true` MUST be accepted only on a `manyToOne` or `oneToOne` relation whose target declares
> `identity`, at most once per entity; on a `subset`, a `manyToMany`, a second owner relation and a
> target without `identity` it MUST be reported as an authoring error (for a target owned by another
> model, at generation). A conforming generator MUST let the regular list, its count and its search
> be narrowed to the rows whose owner relation is the caller's identity record, resolved as the
> personal surface resolves it; a caller whom no identity record maps owns no row, so the narrowed
> list MUST be empty and its count 0. The narrowing MUST apply after the regular surface's role gate
> and MUST NOT widen what it lets through. The regular list page MUST offer the choice between all
> rows and the caller's own, all rows being the default. `owner` MUST NOT scope a write or add a
> surface, and an entity without an owner relation MUST generate byte-identical output.

## DSL index

A new row, after `personal` / `partner`:

| Construct | What it does |
| --- | --- |
| [`owner`](#owner---a-mine-filter-on-the-regular-list) | a to-one to an `identity` entity: the regular list offers All / Mine, narrowed to the caller's own rows; no surface is added |
