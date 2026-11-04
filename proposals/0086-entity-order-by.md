# An entity's default row order - `orderBy` - for the list, the tree and every picker

- **Status:** released in [1.12](../versions/1.12.md)
- **Issue:** https://github.com/IntentFile/intent-specification/issues/147, https://github.com/eclipse-dirigible/dirigible/issues/7727
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7739

## The problem

The format says **which** columns a list shows (`list:`, proposal 0065) and in what order the
**form's** controls run (`order:`), and nothing about the order of the **rows**. A chart of accounts
comes back in whatever order the database returns it: in the list, in the `hierarchy:` tree and in
every picker that targets it, with the account number nowhere in evidence. The list sorts only when
the user clicks a column header.

1.11's entity-level attribute table has `order:` ("sequences form controls and list columns") and
`list:` ("the exact column set and order of the entity's lists"); neither orders rows, and there is
no other key that does.

## The proposed shape

```yaml
entities:
  - name: Account
    orderBy: [number]                                # ascending; a single term needs no list
    hierarchy: Parent
  - name: SalesInvoice
    orderBy: [{ field: date, dir: desc }, number]    # newest first, then by number
```

## Expected behaviour

- `orderBy:` is the entity's default row order. A term is a field or a to-one relation of the
  entity, written as a bare name or as `{ field, dir }`; `dir` is `asc` or `desc`, default `asc`. A
  to-one relation orders by its key.
- The order is applied by the query that reads the rows, so a page of a paged, filtered (by a master
  record) or searched list is a slice of one ordered sequence - never a sort of whichever rows the
  page happened to return. The scoped (personal / partner) lists read in the same order.
- The same order reaches the tree a `hierarchy:` entity renders (siblings in that order) and every
  picker whose target declares it. A picker over a target without `orderBy:` keeps its usual sort by
  display text.
- `orderBy:` is only the default: a header click still re-sorts the list.

## Edge rules

- Refused when the intent is read: a name the entity does not declare, a term listed twice, and a
  direction that is neither `asc` nor `desc`.
- A single term may be written without the list.
- An entity without `orderBy:` lists, and its pickers sort, as before.

## Prior art / workarounds

None within the format: the reader clicked a header on every visit, and the tree and the pickers
had no header to click. A hand-edited query in the generated list was lost on the next generation.
Reference implementation: https://github.com/eclipse-dirigible/dirigible/pull/7739.

## Specification text

**Anchor:** Entities & fields > Control order, appended after the `list:` rules, plus a row in the
entity-level attribute table after `list:`.

`order:` and `list:` sequence the controls and the columns; neither says in what order the **rows**
come. `orderBy:` is the entity's default row order:

```yaml
- name: Account
  orderBy: [number]                                # ascending; a single term needs no list
- name: SalesInvoice
  orderBy: [{ field: date, dir: desc }, number]    # newest first, then by number
```

A term is a field or a to-one relation of the entity - a relation orders by its key - written as a
bare name, ascending, or as `{ field, dir }` with `dir: asc | desc`. The order holds wherever the
entity's rows are listed: the list, the tree of a [`hierarchy:`](#hierarchy--leafonly--tree-entities)
entity and every picker that targets the entity. It is a default, not a constraint: a reader who
sorts the list by a column header sees that sort.

> **Normative.**
> When an entity declares `orderBy:`, a conforming generator MUST apply the declared terms, in the
> declared sequence and direction, in the query that reads the entity's rows for its lists, so that
> every page of a paged, master-filtered or searched list is a slice of one ordered sequence; sorting
> the rows of a page after they are read does not conform. The siblings of a `hierarchy:` tree and the
> options of every picker that targets the entity MUST follow the same order; a picker over a target
> without `orderBy:` keeps the order it has without one. A user-chosen sort MUST take precedence over
> `orderBy:` for as long as it is chosen. Each term MUST be a field or a to-one relation of the entity,
> and a to-one relation MUST order by its key. A single term MAY be written as a scalar. A name the
> entity does not declare, a term listed twice and a `dir` other than `asc` or `desc` MUST be reported
> as authoring errors. An entity without `orderBy:` MUST generate as it did before.

| Attribute | Effect |
| --- | --- |
| `orderBy:` | the default row order of the entity's lists, its tree and every picker that targets it (see [Control order](#control-order)) |

## DSL index

| Construct | What it does |
| --- | --- |
| [`orderBy`](#control-order) | the entity's default row order - applied by the query, so every page is a slice of one sequence - reaching the list, the tree and every picker that targets it |
