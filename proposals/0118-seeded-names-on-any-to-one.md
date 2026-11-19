# A seeded name resolves on any to-one relation of the record

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/163, https://github.com/eclipse-dirigible/dirigible/issues/8015, https://github.com/eclipse-dirigible/dirigible/issues/8057
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8016, https://github.com/eclipse-dirigible/dirigible/pull/8059

## The problem

"Status references - name, not number" lets the seeded name stand for a **status** at the sites it
lists, and 0076 extends that to a decision on any to-one relation of the trigger entity. A record
classified by another nomenclature - a request's `type`, OPEN or CLOSE - is as often what a
condition tests, and at the other condition sites the name was resolved on the status relation
only:

- a trigger guard `when: "type == CLOSE"` compared the integer key with the text `CLOSE`, never
  held, and the process never started;
- a schedule's `{ field: type, op: eq, value: CLOSE }` never matched a row;
- a check's `when` and a `visibleWhen` refused `type == CLOSE` as "not a value of that type", so
  the model had to write the positional id `type == 2`, which a seed inserted mid-nomenclature
  silently retargets - the defect the section exists to prevent.

The first two passed generation and failed silently at runtime. This is a clarification of where
the existing rule applies, not a new key.

## The proposed shape

No new key. The example that must hold:

```yaml
entities:
  - name: Request
    relations:
      - { name: type, kind: manyToOne, to: RequestType }     # seeded OPEN, CLOSE
    checks:
      - { kind: requiredWhen, field: note, when: "type == CLOSE", message: "Say why it closes" }
processes:
  - name: close
    trigger: { onCreate: Request, when: "type == CLOSE" }
    # steps as usual
schedules:
  - name: remindOpen
    cron: "0 0 9 * * ?"
    entity: Request
    where: [ { field: type, op: eq, value: OPEN } ]
    notify: { to: desk@example.com, subject: "Request {id} is open", body: "Please follow up." }
```

Each `CLOSE` and `OPEN` resolves to the seed id of that name in `RequestType`.

## Expected behaviour

A seeded name may stand for a value of any to-one relation of the record, each resolved against
its own relation's target, in:

- a process trigger's `when` (scalar and list form);
- a `where` condition: `schedules[].where`, a create-from's `items.where`, `rollups[].where`;
- a check's `when` (`requiredWhen`, `forbidWhen`);
- a field's or a composition child's `visibleWhen`.

A composition child's entity-level `visibleWhen` resolves against its master's relations, as its
status terms already do. A `forbidWhen` one-hop term keeps its existing resolution against the
relation target's nomenclature.

## Edge rules

- Only equality (`==`, `!=`, `eq`, `ne`) is accepted with a name: a name has no ordering, and an
  ordering comparison against one is an authoring error.
- An unknown name is an authoring error naming the site and the nomenclature.
- A name against a target owned by another model is an authoring error directing the author to the
  numeric id, as for a status.
- Numeric ids, fields and `null` tests are left as written.
- A misspelt status in a guard that also names another relation is reported once.
- The `when` guards of `notifications`, `generates`, `postings` and `wait` are not widened here.

## Prior art / workarounds

The positional seed id. Reference implementations:
https://github.com/eclipse-dirigible/dirigible/pull/8016 and
https://github.com/eclipse-dirigible/dirigible/pull/8059.

## Specification text

**Anchor:** Data, seeds & naming > seeds > Status references - name, not number, after its
normative block.

The same holds for any to-one relation of the record that targets a seeded nomenclature, not only
its status. A request classified OPEN or CLOSE is tested as `type == CLOSE`, resolved against the
seeds of the relation's own target.

> **Normative.**
> A seeded name MAY stand for a value of any to-one relation of the record, not only its status, in
> a trigger's `when`, a `where` condition, a check's `when` (`requiredWhen`, `forbidWhen`) and a
> `visibleWhen`; each resolves against its own relation's target. A composition child's
> entity-level `visibleWhen` resolves against its master's relations. Only equality (`==`, `!=`,
> `eq`, `ne`) is accepted with a name - a name has no ordering. An unknown name and a cross-model
> target MUST be refused at generation; numeric ids, fields and `null` tests are left as written.

## DSL index

No new row; the existing rows are unchanged.
