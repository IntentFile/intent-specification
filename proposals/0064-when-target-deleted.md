# What deleting a referenced record does to the records pointing at it - `whenTargetDeleted`

- **Status:** released in [1.10](../versions/1.10.md)
- **Issue:** https://github.com/IntentFile/intent-specification/issues/100, https://github.com/eclipse-dirigible/dirigible/issues/7547
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7573, https://github.com/eclipse-dirigible/dirigible/pull/7657

## The problem

The format says what deleting a master does to its composition children
([0033](0033-master-delete-and-its-composition-children.md)) and nothing about a plain reference. A
plain `manyToOne` - an invoice's customer, an employee's department - gives the referenced row no
delete protection: deleting the customer succeeds, and every invoice keeps a key that points at
nothing. Referential integrity is a business-layer rule in this format, not a database constraint, so
nothing underneath catches it either.

Three answers exist, and a domain needs all three in different places: refuse the delete while
anything references the row; clear the reference (an employee leaves, the tickets she was assigned
become unassigned); delete the referencing rows with it (a scheduled reminder of a deleted document).

## The proposed shape

```yaml
relations:
  - { name: customer, kind: manyToOne, to: Customer }                                  # restrict - the default
  - { name: assignee, kind: manyToOne, to: Employee,     whenTargetDeleted: nullify }
  - { name: invoice,  kind: manyToOne, to: SalesInvoice, whenTargetDeleted: cascade }
```

## Expected behaviour

Every plain to-one carries a rule; `restrict` is the default, also when the key is absent, because a
dangling reference is never a valid state.

- `restrict` - the target's delete is refused as a validation outcome naming the referencing entity
  and the count of its rows.
- `nullify` - the referencing rows' key is cleared, in the same unit of work as the delete.
- `cascade` - the referencing rows are deleted with the target, each through its own delete path, so
  their own rules (their children, their own `whenTargetDeleted` relations) apply in turn.

The rule rides the **referencing** relation - the target's model cannot know who points at it - and
holds the same way whether the target is in the same model or in another one.

## Edge rules

- `nullify` on a `required` relation is an authoring error: the cleared row would violate its own
  shape.
- A `composition: true` relation takes no `whenTargetDeleted`: a child's fate is 0033's rule.
- The relation to a `function: EntityStatus` nomenclature takes none either; a status row is seeded,
  not deleted.
- A target referenced through several relations is governed by each: one `restrict` refuses the
  delete whatever the others say.
- A system or workflow delete follows the same rules; the rule is about the data, not about who acts.

## Prior art / workarounds

A hand-written guard in the target's custom code, which has to enumerate every referencing entity in
every model and is wrong the day a new one is added. Reference implementation: https://github.com/eclipse-dirigible/dirigible/pull/7573
(`restrict`), https://github.com/eclipse-dirigible/dirigible/pull/7657 (`nullify` / `cascade`, cross-model).

## Specification text

**Anchor:** Relations & multi-model > Relation attributes, as a new bullet after `dependsOn`, and a
row in the relation attribute table.

- **`whenTargetDeleted: restrict | nullify | cascade`** says what deleting the referenced record does
  to the rows of this entity that point at it. `restrict` (the default, also when the key is absent)
  refuses the delete while any row references the target; `nullify` clears the reference; `cascade`
  deletes the referencing rows with it. The rule is declared on the referencing relation and holds
  across models.

> **Normative.**
> A conforming generator MUST refuse the delete of a record that is referenced through a relation
> declaring or defaulting to `restrict`, as a validation outcome naming the referencing entity and
> the number of its rows. Under `nullify` it MUST clear the referencing rows' key, and under
> `cascade` it MUST delete the referencing rows through their own delete path so that their rules
> apply in turn - both in the same unit of work as the target's delete, so a failure leaves the
> target in place. The rule MUST hold whether or not the target is owned by the declaring model, and
> for system and workflow deletes alike. A generator MUST reject `nullify` on a `required` relation,
> and `whenTargetDeleted` on a composition relation or on a `function: EntityStatus` relation. A
> file that declares no `whenTargetDeleted` MUST behave as if every plain to-one declared `restrict`.

## DSL index

| Construct | What it does |
| --- | --- |
| [`whenTargetDeleted`](#relation-attributes) | on a plain to-one: what deleting the referenced record does to the referencing rows - `restrict` (default), `nullify`, `cascade`; same-model and cross-model |
