# A `forbidWhen` refuses the delete of the row it guards, and a term may test presence

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/105, https://github.com/eclipse-dirigible/dirigible/issues/7372, https://github.com/eclipse-dirigible/dirigible/issues/7555, https://github.com/eclipse-dirigible/dirigible/issues/7509
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7417, https://github.com/eclipse-dirigible/dirigible/pull/7583, https://github.com/eclipse-dirigible/dirigible/pull/7528
- **Amends:** [0047 - checks: forbidWhen](0047-checks-forbid-when.md)

## The problem

0047 is normative that "a `forbidWhen` does not refuse a delete; delete under a master's status is
`immutableWhen` / `locksWithMaster`'s rule". Two cases that rule cannot reach showed up within a week
of its release.

A payroll entry swept into a payslip is part of that payslip; deleting it after the sweep leaves the
payslip's figures orphaned. The entry has no status, so `immutableWhen` has nothing to key on; the
payslip is not its composition master, so `locksWithMaster` does not apply; and the rule is "while
the entry's `Payslip` relation is set" - a presence test, which 0047's grammar does not have.

And the rule 0047 was written for - no allocation while the invoice is PAID - is as true of removing
an allocation as of adding one: removing it un-pays the invoice from the wrong door.

## The proposed shape

```yaml
- name: PayrollEntry
  checks:
    - { kind: forbidWhen, when: "Payslip != null",
        message: "An entry swept into a payslip is part of it" }
- name: SalesInvoiceCustomerPayment
  checks:
    - { kind: forbidWhen, when: "SalesInvoice.Status == PAID",
        message: "The invoice is already paid" }
```

## Expected behaviour

A `forbidWhen` covers the create, the update **and the delete** of the row it is declared on. On a
delete the condition is judged against the stored row.

A term may be `<Property> == null` or `<Property> != null`, on a field or a to-one of the record or
on a one-hop `Relation.field`: `null` is the presence test and is never read as a string. A to-one
term compared by value (`Relation == <id or seeded name>`) reads the relation's key, as
`requiredWhen`'s reader does.

## Edge rules

- The master-detail affordance hiding of 0047 is unchanged and now covers the delete affordance for
  the reason it already hid it.
- An unset relation still reads as no value in a value comparison: `== value` does not hold, `!=
  value` does; `== null` holds, `!= null` does not.
- A gated `forbidWhen` (`status:`) is judged on a delete against the status the stored row carries.
- 0047's "does not refuse a delete" sentences, in the prose and in the normative block, are
  withdrawn by this proposal; the rest of 0047 stands.

## Prior art / workarounds

A hand-written delete guard. Reference implementation: https://github.com/eclipse-dirigible/dirigible/pull/7417 (the delete),
https://github.com/eclipse-dirigible/dirigible/pull/7583 (the presence test), https://github.com/eclipse-dirigible/dirigible/pull/7528 (a to-one term by value).

## Specification text

**Anchor:** Entities & fields > checks > kind: forbidWhen - a write refused while a condition holds.
The first sentence becomes "refuses a create, update or delete of the row while its condition holds";
the normative sentence "A `forbidWhen` does not refuse a delete; delete under a master's status is
`immutableWhen` / `locksWithMaster`'s rule." is removed; the following is appended to the prose and
the normative block.

A term may test whether a value is set at all: `<Property> != null` holds while the field or the
to-one is set, `<Property> == null` while it is not. A to-one compared against a value reads its key.

```yaml
- name: PayrollEntry
  checks:
    - { kind: forbidWhen, when: "Payslip != null", message: "An entry swept into a payslip is part of it" }
```

> **Normative.**
> A conforming generator MUST refuse the delete of the row while every term of `when` holds against
> the stored row, with `message`, on every surface it generates for user deletes, as it refuses a
> create or update; a `status:` gate is judged against the status the stored row carries. A term
> `<Property> == null` MUST hold exactly when the property has no value and `<Property> != null`
> exactly when it has one; `null` MUST NOT be read as a string literal. A to-one named on the left
> of a value comparison MUST be compared by its key. The master-detail affordance hiding MUST include
> the delete affordance under the same condition.

## DSL index

| Construct | What it does |
| --- | --- |
| [`checks: kind: forbidWhen`](#kind-forbidwhen--a-write-refused-while-a-condition-holds) | a create, update or delete refused while a condition over the record or one hop holds; `== null` / `!= null` test presence |
