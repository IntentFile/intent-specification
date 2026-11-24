# A declared reopen whose target another model owns

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7954
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8889
- **Builds on:** [0024 - a retired target stops blocking its source](0024-supersede-a-retired-target.md), [0025 - a declared reopen](0025-declared-reopen-on-retire.md)

## The problem

The commonest create-from in a modular business application bills a document of one module into
another module's invoice: a project milestone, a month of timesheets. The milestone moves to
INVOICED when the invoice is made, and only from REACHED may it be billed.

When the invoice is voided, the milestone must return to REACHED so it can be billed again. That
is `sourceStatusOnRetire`, and it is rejected today whenever the target belongs to another model,
on the ground that the target's nomenclature is seeded there and its `stage:` classification is not
resolvable where the rule is declared. The milestone stays INVOICED for good, the from-status guard
refuses every new billing of it, and its amount can never be invoiced again.

The invoicing module cannot declare the reopen instead: it would have to use the milestone's model,
which already uses it.

## The proposed shape

```yaml
uses:
  - { model: sales-invoices }

generates:
  - name: invoice-from-milestone
    from: Milestone
    to: SalesInvoice
    uses: sales-invoices
    map:
      reference: number
      milestone: id             # the back-reference: a plain integer field of the invoice
    fromStatus: [REACHED]
    sourceStatus: INVOICED
    sourceStatusOnRetire: REACHED   # voiding the invoice returns the milestone
```

The owner model classifies its own nomenclature, as it does for any reader:

```yaml
seeds:
  - name: sales-invoice-statuses
    entity: SalesInvoiceStatus
    rows:
      - { id: 3, name: ISSUED,    stage: live }
      - { id: 8, name: CANCELLED, stage: cancelled }
      - { id: 9, name: VOIDED,    stage: void }
```

## Expected behaviour

- **The rule is accepted** when its target belongs to another model. Which statuses retire the
  target is read from the owner model's classification of the target's nomenclature, the same
  classification a reader of that model already relies on.
- **Retiring the target returns the source** exactly as for a target of the same model: the source
  is moved to `sourceStatusOnRetire` while it stands at `sourceStatus` and no target of it still
  counts, and the move is announced on the source's transition channel.
- **The return follows the retirement, never precedes it.** A retirement the target refuses is not
  a retirement: the source stays where it stands. A source whose return is announced has a target
  whose retirement took effect.
- **The at-most-once guard** of an event-driven rule over such a target steps over a retired one, by
  the same classification, so the re-fired trigger mints the replacement.
- **The owner model is unaware.** Nothing is declared in it beyond its own `stage:` classification,
  and it does not name the source's model.

## Edge rules

- Rejected when the owner model's target carries no `function: EntityStatus` relation, or when the
  owner classifies no seed row of its nomenclature `cancelled` or `void`. Both are known only once
  the owner model is resolved, so a generator reports them where it resolves it.
- The back-reference rule is unchanged: the `map` copies the source's key onto a property of the
  target. A target in another model cannot hold a relation to a model that uses it, so that
  property is a plain value field there.
- Every other rule of 0025 applies unchanged: `sourceStatus` is required and must differ, an
  appending rule is rejected, and a source lifecycle must declare the edge back.
- A file that declares no `sourceStatusOnRetire` regenerates unchanged. A file whose event-driven
  rule targets another model now steps over a retired target where the owner classifies its
  nomenclature, as the same rule over a local target always did.

## Prior art / workarounds

A person moved the source back by hand, which its lifecycle usually forbids, or the target module
was given a reference to the source module, which closes a dependency cycle between the two.

## Specification text

**Anchor:** Declarative glue > generates — create-from > Event-driven creation > A declared reopen —
`sourceStatusOnRetire`, replacing the clause "or when the target belongs to another model (its
nomenclature is seeded there)" of the normative block, and the last sentence of the informative
paragraph after the normative block of "A retired target stops blocking its source".

In the normative block of "A declared reopen", the rejection reads:

> It MUST be rejected when the target's retirement cannot be recognised: when the target carries no
> `function: EntityStatus` relation, or when its nomenclature carries no `cancelled` or `void`
> `stage:` classification.

and the following is added after it:

A target may belong to another model. Its nomenclature is seeded and classified there, and the
reopen reads that classification:

```yaml
generates:
  - name: invoice-from-milestone
    from: Milestone
    to: SalesInvoice
    uses: sales-invoices
    map: { reference: number, milestone: id }
    sourceStatus: INVOICED
    sourceStatusOnRetire: REACHED
```

> **Normative.**
> When the target of a create-from belongs to another model, a generator MUST resolve whether a
> status retires it from the owner model's `stage:` classification of the target's nomenclature,
> and MUST reject a `sourceStatusOnRetire` whose owner target carries no `function: EntityStatus`
> relation or whose owner classifies no seed row `cancelled` or `void`. The reopen MUST be produced
> on the side of the source's model, which already depends on the owner; the owner model MUST NOT
> be required to declare anything about it. The source MUST be moved only after the target's
> retirement has taken effect: a retirement that is refused MUST leave the source unchanged. The
> at-most-once guard of an event-driven create-from over such a target MUST read the same
> classification.

The informative paragraph after the normative block of "A retired target stops blocking its source"
ends instead:

A target owned by another model is classified in its owner model, and the guard reads that
classification as it reads a local one.

## DSL index

No new construct. The row of `generates.sourceStatusOnRetire` reads:

| Construct | What it does |
| --- | --- |
| [`generates.sourceStatusOnRetire`](#a-declared-reopen--sourcestatusonretire) | the completion hook's inverse: retiring the generated target - of this model or of another - returns the source, and the trigger re-fires |
