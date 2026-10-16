# A master's lock reaches its whole composition chain

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/110, https://github.com/eclipse-dirigible/dirigible/issues/7550
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7571

## The problem

The specification says an entity's immutability "covers its composition children". Read as the direct
children only - which is how it was first implemented - a POSTED payroll run locks its payslips while
the payslips' **lines** stay editable: the payslip declares no lock of its own, so its lines inherit
nothing, and the run's figures can be changed from two levels down. `forbidWhen` hops one relation
and cannot close the gap from the line.

This is a clarification of meaning, not a new key, but it changes what a conforming generator must
refuse, so it is recorded as a proposal.

## The proposed shape

No new key. The example that must hold:

```yaml
- name: PayrollRun
  immutableWhen: "Status == POSTED"
- name: Payslip
  relations: [ { name: run, kind: manyToOne, to: PayrollRun, composition: true } ]
- name: PayslipLine
  relations: [ { name: payslip, kind: manyToOne, to: Payslip, composition: true } ]
```

A POSTED run refuses a user write of a payslip **and** of a payslip line.

## Expected behaviour

The lock descends the whole composition chain below a locked master; each level inherits the nearest
lock above it, through masters that declare no lock of their own. The descent stops at a child that
declares `locksWithMaster: false`: it outlives the lock above it, and so does everything composed
into it.

## Edge rules

- A child with its own lock (`immutableWhen`, `immutable`, a period lock) is locked by either: its
  own, or any ancestor's.
- The period lock (0028) descends the same way.
- System and workflow writes are unaffected at every level, as for the direct children.

## Prior art / workarounds

A `forbidWhen` per level, each reading one hop up - which cannot be written for the grandchild.
Reference implementation: https://github.com/eclipse-dirigible/dirigible/pull/7571.

## Specification text

**Anchor:** Entities & fields > lifecycle - the legal status graph, the paragraph "An entity's
immutability also covers its composition children" and its normative block.

An entity's immutability also covers its **composition children, at every depth**: a child inherits
the nearest lock above it, through masters that declare none of their own, down to a child declaring
`locksWithMaster: false`, which outlives the lock together with everything composed into it.

> **Normative.**
> A generator MUST refuse a user create, update or delete of any record in the composition chain
> below a currently immutable master - a child, a child's child, and so on - unless the record, or a
> master between it and the locked one, declares `locksWithMaster: false`. The refusal MUST cover
> every user surface it generates and MUST NOT extend to system / workflow writes.

## DSL index

No new row; the `immutableWhen` / `locksWithMaster` rows are unchanged.
