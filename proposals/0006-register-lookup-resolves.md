# `resolves` - fill a relation from a register valid on a date


- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/6712
- **Discussion:** https://github.com/IntentFile/intent-specification/pull/15

## The gap

A very common enterprise shape has no declarative form in the format today: **resolve a relation by consulting a register with a validity period**. The register rows say "X applied to Y from A to B"; an incoming record carries the match key(s) and a date, and a to-one must be set from the row whose period covers that date.

The same pattern under many names:

- a fine carries a vehicle and a violation date; the **driver** comes from a vehicle-assignment register;
- a price comes from the **price list** valid on the order date;
- a rate from the **contract** in force on the booking date;
- an approver from an **org assignment** on the request date.

Nothing reaches it: `dependsOn` is an authoring-time copy matched by equality, a `decision` condition is a single comparison, and a `setField` step writes a constant. Every application hand-writes the same code — query the register, apply the between-dates predicate, classify the result.

## What this adds

A `resolves` section in the declarative-glue chapter of 1.2:

```yaml
resolves:
  - name: identifyDriver
    event: { onCreate: Fine }
    set: driver
    from: VehicleAssignment
    match: { vehicle: vehicle }
    between: { start: validFrom, end: validTo, value: violationAt }
    outcome: resolution
    found:     { setStatus: IDENTIFIED }
    notFound:  { setStatus: UNRESOLVED }
    ambiguous: { setStatus: UNRESOLVED }
```

The normative content is deliberately about the two properties that make this a construct rather than a query helper:

1. **All three outcomes are distinguished.** Exactly one covering row fills the relation; no covering row and more than one covering row both leave it unset, and a conforming generator MUST NOT choose between candidate rows. An automation that silently picks one of two candidates is worse than none.
2. **The attempt is observable.** `outcome:` stamps `found` / `notFound` / `ambiguous` into a string field, so the unresolved records form a worklist a person can finish and a process `decision` can branch on the result.

Plus the constraints that keep it unambiguous: the register MUST carry exactly one to-one relation to the entity `set` points at (that relation is the copied value — zero or two is a modelling ambiguity, not something to guess); a record that already carries the relation MUST be skipped, so a manual correction is never overwritten; and the relation, outcome and status MUST be written as one targeted update leaving every other column alone. Period bounds may be open on either side, the end is inclusive, and a date-only bound covers its whole day.

Also updates the 1.2 change list, the construct index and the README version row.

Implemented in Eclipse Dirigible (eclipse-dirigible/dirigible#6712, PR eclipse-dirigible/dirigible#6732) — the implementation and this text were written together.

## Specification text

The prose below is what a release folds into the next version document, at the anchors
given. It was written against `versions/1.2.md` and is carried here unchanged.

**Anchor:** The Intent File Specification > Version 1.2

- **[`resolves`](#resolves--fill-a-relation-from-a-register-valid-on-a-date)** — fill a to-one from the register row valid on a date the record carries, with `found` / `notFound` / `ambiguous` as three first-class, observable outcomes.

**Anchor:** Declarative glue > posts — derived rows on an event

### resolves — fill a relation from a register valid on a date

Set a to-one from the row of a **register** whose validity period covers a date the record carries. The register says "X applied to Y from A to B" — a vehicle assignment, a price list, a contract in force, an org assignment — and the record carries the match key(s) and the date:

```yaml
resolves:
  - name: identifyDriver
    event: { onCreate: Fine }               # onCreate or onUpdate, optional `when` guard
    set: driver                             # the to-one of Fine this fills
    from: VehicleAssignment                 # the register
    match: { vehicle: vehicle }             # register property <- record property (one or more)
    between: { start: validFrom, end: validTo, value: violationAt }
    outcome: resolution                     # optional string field: found / notFound / ambiguous
    found:     { setStatus: IDENTIFIED }
    notFound:  { setStatus: UNRESOLVED }
    ambiguous: { setStatus: UNRESOLVED }
```

Nothing else in the format reaches this shape: [`dependsOn`](#relations) is an authoring-time copy matched by equality, a [`decision`](#processes) condition is a single comparison, and a `setField` step writes a constant. Without it every application hand-writes the same query-and-classify code.

**Normative.** A lookup MUST declare exactly one of `onCreate` / `onUpdate` naming a declared entity; `onDelete` MUST be rejected, since there is no record left to fill. `set` MUST name a to-one relation of that entity, `from` a declared register entity, and `match` at least one pair whose left side is a property of the register and whose right side a property of the record. `between.value` MUST name a date field of the record; `between.start` and `between.end` name date fields of the register and MAY each be omitted, in which case that side of the period is open. The end of a period is **inclusive**, and a bound expressed as a date (rather than an instant) covers its whole day.

**Normative.** The register MUST carry exactly one to-one relation to the entity `set` points at; that relation is the value the lookup copies. Zero or more than one MUST be rejected — a register offering a choice of columns to copy is a modelling ambiguity, and guessing one would defeat the construct's purpose.

**Normative.** All three outcomes are first-class and MUST be distinguished. Exactly one covering row fills the relation. No covering row (`notFound`) and more than one covering row (`ambiguous`) MUST both leave the relation unset: a conforming generator MUST NOT choose between candidate rows. Each outcome MAY carry a `setStatus` routing the record, which requires the record to declare a `function: EntityStatus` relation and accepts a [status name](#status-references--name-not-number) as well as an id.

**Normative.** The attempt MUST be observable. When `outcome` names a `string` field of the record, that field MUST be stamped with `found`, `notFound` or `ambiguous`, so unresolved records form a filterable worklist a person can finish and a process [`decision`](#processes) can branch on the result. A conforming generator SHOULD additionally log the keys and the date it checked.

**Normative.** A record that already carries the relation MUST be skipped, so a manual correction is never overwritten and a re-delivered event is a no-op. The resolved relation, the outcome and the status MUST be written as one targeted update of those columns only, leaving every other column of the record — and any concurrent write to it — untouched.

**Anchor:** Appendix A: DSL index

| [`resolves`](#resolves--fill-a-relation-from-a-register-valid-on-a-date) | fill a to-one from the register row valid on the record's date |
