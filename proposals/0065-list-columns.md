# The list's own column set and order - `list`

- **Status:** released in [1.11](../versions/1.11.md)
- **Issue:** https://github.com/IntentFile/intent-specification/issues/101, https://github.com/eclipse-dirigible/dirigible/issues/7614
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7615

## The problem

`order:` sequences the form inputs, the list columns and the detail rows **together**, and the list
shows every property marked major in that order. A document wants two orders: its form in data-entry
order (number, customer, date, due date, currency, notes) and its list in scanning order (number, date,
customer, total, balance, status). With one knob, fixing the list breaks the form, and the only way to
drop a column from the list drops it from the form too.

## The proposed shape

```yaml
- name: SalesInvoice
  order: [Number, Customer, Date, Due, Currency, Notes]           # the form and the detail rows
  list:  [Number, Date, Customer, Total, Balance, Due, Status]    # the list
```

## Expected behaviour

`list:` is the exact column set and order of the entity's generated lists - the main list and every
embedded list that shows the entity's rows. Without it nothing changes: the list keeps following
`order:` and the major properties. With it, `list:` overrides the major marking for the list only;
the form and the detail rows keep following `order:`.

## Edge rules

- Every name must be a field, a to-one relation (`manyToOne`, `oneToOne`, `subset`), or the stored
  display name an entity-level `label:` generates. A to-many relation has no column and is refused.
- No blank entry, no name twice.
- A name that is neither in `order:` nor rendered on the form is still a valid list column: the list
  may show a derived or system-owned value the form does not edit.

## Prior art / workarounds

A hand-edited generated list, lost on the next generation. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/7615.

## Specification text

**Anchor:** Entities & fields > Control order (append), plus a row in the entity-level attribute
table after `order:`.

A list is read in a different order than a form is filled. `list:` names the list's own columns:

```yaml
- name: SalesInvoice
  order: [Number, Customer, Date, Due, Currency, Notes]
  list:  [Number, Date, Customer, Total, Balance, Due, Status]
```

With `list:` the generated lists show exactly these columns in this order; the form and the detail
rows keep following `order:`. Without it the list follows `order:` as before.

> **Normative.**
> When an entity declares `list:`, a conforming generator MUST render exactly the named properties
> as the columns of every list of that entity, in the declared order, and MUST leave the form and
> the detail rows governed by `order:`. Each name MUST be a field, a to-one relation or the entity's
> stored display name; a to-many relation, a blank entry, an unknown name and a repeated name MUST be
> reported as authoring errors. An entity without `list:` MUST generate as it did before.

| Attribute | Effect |
| --- | --- |
| `list:` | the exact column set and order of the entity's lists, decoupled from `order:` |

## DSL index

| Construct | What it does |
| --- | --- |
| [`list`](#control-order) | the generated list's own column set and order, independent of the form's `order:` |
