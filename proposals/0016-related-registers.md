# Related registers - the records that reference an entity


- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/6673
- **Discussion:** https://github.com/IntentFile/intent-specification/pull/29

An entity page shows its own fields, and a document shows its composition items. An entity that is the **target** of associations had no way in the specification to show the records pointing at it — a project-month and its per-employee timesheet lines, a customer and its invoices, an account and its journal entries, a supplier and its purchase orders. Today the answer is a different perspective, reached by filtering.

This adds `related:` to the *Relations & multi-model* chapter (plus its Appendix A row):

```yaml
- name: ProjectTimesheet
  related:
    - entity: EmployeeTimesheet
      model: employee-timesheets
      via: projectTimesheet
      label: Employee Timesheets
      show: [number, employee, totalHours, status]
```

Two points the section argues rather than asserts:

- **The referenced side declares it, and it has to.** Generation is per model and leaf-first, so the model being referenced is generated before — and generally knows nothing about — the models that reference it. A declaration on the referencing side could never reach the page it wants to appear on.
- **It is a window, not an owner.** The listed records belong to their own entity, with their own lifecycle, pages and processes; the register lists them and stops. That is what separates it from a composition child, which *is* edited in place — and why a composition child is rejected here rather than rendered a second time.

The normative rules cover what a generator could otherwise get quietly wrong: an ambiguous source (an invoice naming the same company as both issuer and recipient) must be rejected rather than resolved by guess; a cross-model register must fail loudly rather than come back with no columns; no create/update/delete affordances; and field-visibility rules apply to a register's columns exactly as to the referencing entity's own lists.

Reference implementation, with emission and runtime coverage at both ends: eclipse-dirigible/dirigible#6761.

## Specification text

The prose below is what a release folds into the next version document, at the anchors
given. It was written against `versions/1.2.md` and is carried here unchanged.

**Anchor:** Static option filter - e.g. only stock-tracked products: > Multi-model applications > Generate leaf-first, then publish everything


### related — list the records that reference this entity

An entity page shows its own fields, and a document shows its composition items. An entity that is
the **target** of associations has no way to show the records pointing at it — a project-month and
its per-employee timesheet lines, a customer and its invoices, an account and its journal entries,
a supplier and its purchase orders. `related:` declares that register, on the referenced entity:

```yaml
- name: ProjectTimesheet
  related:
    - entity: EmployeeTimesheet          # the referencing entity
      model: employee-timesheets         # omit when it is declared in this model
      via: projectTimesheet              # omit when it points here exactly once
      label: Employee Timesheets         # omit for the pluralised entity name
      show: [number, employee, totalHours, status]   # omit for the source's own list columns
```

The register renders on the referenced record's page, filtered to that record, and each row opens
the referencing record's own page.

It is a **window, not an owner**. The listed records belong to their own entity — their own
lifecycle, their own pages, their own processes — so the register lists them and stops there. That
is what separates it from a composition child, which *is* edited in place as a detail or
document-items collection.

**Why the referenced side declares it.** Generation is per model and leaf-first: the model being
referenced is generated before, and generally knows nothing about, the models that reference it.
A declaration on the referencing side could therefore never reach the page it wants to appear on.

> **Normative.**
> `entity` is required; a `model:` must be listed in `uses:`.
> `via:` is required when — and only when — the referencing entity reaches this one through more
> than one relation (an invoice naming the same company as both issuer and recipient). A generator
> MUST reject an ambiguous register rather than choose a relation for it.
> Every `show:` name MUST be a field or relation of the referencing entity.
> A generator MUST NOT offer create, update or delete affordances in a register — the referencing
> entity's own pages own those.
> A composition child MUST be rejected rather than listed: it is already rendered as an editable
> collection, and a second read-only rendering of the same rows is two panels over one collection.
> A generator MUST resolve a cross-model register against the owner model, and MUST fail loudly when
> that model cannot be found, rather than emitting a register with no columns.
> Field visibility rules (a role-scoped or otherwise withheld property) apply to a register's columns
> exactly as they apply to the referencing entity's own lists.

**Anchor:** Appendix A: DSL index

| [`related`](#related--list-the-records-that-reference-this-entity) | a read-only register of the records referencing this entity, on its own page |
