# A process decision may branch on a to-one relation of its trigger entity

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/112, https://github.com/eclipse-dirigible/dirigible/issues/7648
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7679

## The problem

The decision step's example branches on a hop (`customer.creditLimit > 10000`), and the prose says a
condition may walk one hop off the trigger entity. It does not say that the trigger entity's **own
to-one relations** are operands. "E-mail it or print it" is modelled as a nomenclature the record
carries as a key - `sendMethod` - and `if: "SendMethod == 1"` reads as outside the grammar. In the
reference implementation it was, until the loader that publishes the record's values to the gateway
was taught to publish its relations too; a decision on one stopped the instance.

## The proposed shape

```yaml
processes:
  - name: dispatch
    trigger: { onTransition: SalesInvoice, when: "Status == ISSUED" }
    steps:
      - { name: how, kind: decision, args: { if: "SendMethod == EMAIL", then: email, else: print } }
      - { name: email, kind: serviceTask, args: { notify: ..., next: done } }
      - { name: print, kind: userTask,    args: { assignee: clerk, form: PrintAndMail } }
      - { name: done,  kind: end }
```

## Expected behaviour

The condition may name the trigger entity's fields and its to-one relations. A relation is compared by
its key; a relation to a classified nomenclature may name the value by its seeded name
(`SendMethod == EMAIL`), as everywhere a status or nomenclature row is referenced. The gateway reads
the stored record, not only what a preceding task form posted, so a decision placed after a task
completed through any surface sees the same values.

## Edge rules

- A one-hop condition (`customer.creditLimit`) is unchanged.
- A seeded name that the relation's nomenclature does not carry is an authoring error.
- A to-many relation is not an operand.

## Prior art / workarounds

A calculated field mirroring the key, or a task form that posts the value as a variable. Reference
implementation: https://github.com/eclipse-dirigible/dirigible/pull/7679.

## Specification text

**Anchor:** Processes & forms > Decision steps, appended.

A decision condition may name the trigger entity's fields and its to-one relations; a relation is
compared by its key, and a relation to a seeded nomenclature may name the value by its seeded name:

```yaml
- { name: how, kind: decision, args: { if: "SendMethod == EMAIL", then: email, else: print } }
```

> **Normative.** A conforming generator MUST make the trigger entity's fields and to-one relations
> available to a decision condition, read from the stored record at the time the gateway is reached,
> and MUST resolve a seeded name on the right-hand side of a relation comparison against that
> relation's nomenclature. A seeded name the nomenclature does not carry, and a to-many relation as
> an operand, MUST be reported as authoring errors.

## DSL index

No new row; the `kind: decision` row gains "its to-one relations, by key or seeded name".
