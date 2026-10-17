# `sourceStatusOnRetire` on a button create-from that declares `sourceStatus`

- **Status:** released in [1.11](../versions/1.11.md)
- **Issue:** https://github.com/IntentFile/intent-specification/issues/106, https://github.com/eclipse-dirigible/dirigible/issues/7647
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7678
- **Amends:** [0025 - a declared reopen on retire](0025-declared-reopen-on-retire.md)

## The problem

0025 makes `sourceStatusOnRetire` conditional on an `event`: "a rule triggered only by a button
carries no at-most-once guard, so nothing blocks a replacement and the button is itself the reissue".
That holds only for a button without a completion hook.

With `sourceStatus:` declared, the hook moves the source off the status the button is offered from,
and the implied `fromStatus` deny ([0032](0032-create-from-guarded-by-source-status.md)) answers the
second click with a conflict. The proforma stands at INVOICED, `immutableWhen` locks it, and when the
invoice is voided nothing in the model can take the proforma back to APPROVED: the hook blocks the
button exactly as the guard blocks the trigger, and the one construct that declares the way back is
refused on it.

## The proposed shape

```yaml
generates:
  - name: invoice-from-proforma
    from: Proforma
    to: Invoice
    label: Create invoice                # a button, no event
    map: { Proforma: id }
    sourceStatus: INVOICED               # the completion hook
    sourceStatusOnRetire: APPROVED       # the declared inverse, now accepted here too
```

## Expected behaviour

`sourceStatusOnRetire` is accepted on a create-from without an `event` when the rule declares
`sourceStatus`. When a target the rule generated is retired and the source still stands at
`sourceStatus`, the source returns to `sourceStatusOnRetire` through its transition channel; the
button, offered from that status, is available again. The same move, by the same rule, in both shapes.

## Edge rules

- A reopen with no `sourceStatus` is still refused: there is nothing to invert.
- Every other refusal of 0025 stands - the same status twice, an appending rule, a target without a
  status relation, a target whose nomenclature has no `cancelled` / `void` stage, a cross-model
  target.
- "No target of that source still satisfies the at-most-once guard" (0025) reads, on a button rule,
  as "no live target of this rule still references the source" - the hook's own condition.
- 0025's normative sentence "It MUST be rejected unless the rule declares an `event`" is withdrawn by
  this proposal.

## Prior art / workarounds

A hand-written status flip on the target's void transition, one per pair. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/7678.

## Specification text

**Anchor:** Declarative glue > generates > Event-driven creation > A declared reopen -
`sourceStatusOnRetire`. The normative sentence beginning "It MUST be rejected unless the rule declares
an `event`" is replaced by:

> It MUST be rejected on a rule that declares neither an `event` nor a `sourceStatus`: such a rule
> keeps no guard and no hook, so nothing blocks a replacement and the button is itself the reissue.
> On a rule with a `sourceStatus` and no `event`, "no target still satisfies the guard" MUST be read
> as "no live target of the rule still references the source", and the return MUST be made under the
> same conditions as for an event-driven rule - the completion hook blocks the button exactly as the
> guard blocks the trigger, and the declared inverse is the same move back.

## DSL index

| Construct | What it does |
| --- | --- |
| [`sourceStatusOnRetire`](#a-declared-reopen--sourcestatusonretire) | the status the source returns to when its generated target is retired - on an event-driven rule, or on a button rule with a `sourceStatus` hook |
