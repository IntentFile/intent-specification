# Enrichment phases: a moment a silent write announces

- **Status:** draft
- **Issue:**
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/issues/6929

## The problem

Not everything a record needs is known when its row is inserted.

An inventory ledger values stock at a moving average. A `StockMovement` row is created when a goods
document posts; a costing listener bound to that create event reads the pool, computes the movement's
`costValue`, and writes it back. It has to write it back **without** raising an update event: an
enrichment is not a user's edit, and publishing one would re-fire every consumer that reacts to an
edit, including the enrichment listener's own siblings.

So the enrichment is silent, and the natural cost-of-sales posting cannot be expressed:

```yaml
postings:
  - name: cogsPosting
    event: { onCreate: StockMovement }        # the only moment there is
    items:
      - { Account: rule(costOfSalesAccount), debit: "CostValue" }
```

The posting listens to the same create event as the costing listener, and nothing orders the two.
The posting may therefore read the row **before** the cost is written, and produce a perfectly
balanced journal entry for a null amount. The parse succeeds, the generation succeeds, the code
compiles, the deployment is green, and the ledger is wrong. Nothing anywhere states that the value is
not ready yet.

The shape is general. Any enrichment a listener computes after the insert has it: a snapshot column
copied from a register, an identifier fetched from an external system, a classification derived from
a lookup. The value exists, the consumer of it is declarative, and there is no moment to bind.

## The proposed shape

An entity declares the enrichment moments it announces:

```yaml
entities:
  - name: StockMovement
    phases: [costed]
    fields:
      - { name: id,        type: integer, primaryKey: true, generated: true }
      - { name: quantity,  type: decimal, precision: 18, scale: 3 }
      - { name: costValue, type: decimal, precision: 18, scale: 2 }
```

A glue consumer binds one:

```yaml
postings:
  - name: cogsPosting
    event: { onPhase: StockMovement, phase: costed }
    creates: JournalEntry
    backReference: StockMovement
    rule: { entity: PostingRule, match: { documentType: "Goods Issue" } }
    items:
      - { Account: rule(costOfSalesAccount), debit: "CostValue" }
      - { Account: rule(inventoryAccount),   credit: "CostValue" }
```

## Expected behaviour

A declared phase is a channel of the entity, distinct from its create, update, delete and status
channels.

The generated data-access layer of an entity that declares `phases:` exposes, per declared phase, an
operation that **applies a set of column values and announces that phase in the same write**. The
enriching listener calls it instead of a plain silent write. Because the values and the announcement
are one write, a consumer of the phase can never observe the announcement without the values, and can
never observe the values without an announcement.

A consumer that binds `onPhase` receives the record exactly as it receives one on any other axis: the
same recipient paths, the same placeholder interpolation, the same optional `when:` guard, the same
payload shape. Only the moment differs.

## Edge rules

- A phase name is a lower-camel identifier. It names both a generated operation and a channel.
- A phase may not be named after a channel the platform itself publishes (an update, a delete, a
  status transition, or any other reserved channel of the implementation). Announcing one would
  re-fire that channel's consumers.
- A phase may not be declared twice on one entity.
- `onPhase` requires a sibling `phase:` naming one of the bound entity's declared phases. A binding
  naming an undeclared phase is rejected: it would bind a channel nothing publishes to, and the
  consumer would simply never fire.
- A `phase:` key on a binding of any other axis is rejected rather than ignored, since ignoring it
  leaves the consumer bound to the moment it was trying to avoid.
- The `when:` guard is optional on `onPhase`. A phase already names one moment, where a status
  transition is any status write.
- A binding declares exactly one axis: `onPhase` alongside another `on*` key is rejected.
- When the bound entity belongs to another model, its phases are declared in that model, so the name
  cannot be resolved from the consumer's side. This is the same limit a cross-model status
  nomenclature has.
- An announcement carries values by definition. Announcing a phase with nothing to write is not a
  moment that happened.

## Prior art / workarounds

Today the only way to act on an enriched value is to move the consumer into the hand-written
enrichment listener itself: the costing listener computes the cost and then also writes the journal
entry. That discards every guarantee the declarative construct carries (idempotence through the
back-reference, the rule-row resolution, the unposted worklist on a missing account) and puts ledger
logic in a costing routine.

The two alternatives considered and rejected:

- **A global ordering tier** ("enrichment listeners run before consumers"). Messaging delivery to
  independent subscribers is concurrent by construction; an ordering contract across independent
  consumers is one an implementation cannot keep, and one that would quietly stop holding under load.
- **Documenting the race.** It leaves the automation inexpressible, and the failure it warns about is
  silent, so the warning has to be read by the one author who did not think to look.

A phase needs neither: it is an explicit channel, declared where the moment exists and bound where it
is consumed.

## Specification text

**Anchor:** Entities > phases; Declarative glue > The event axis

### phases

An entity may declare the enrichment moments it announces:

```yaml
entities:
  - name: StockMovement
    phases: [costed]
```

An enrichment is a value written to a record after its insert by a listener reacting to that insert.
It is not a user edit and must not be published as one, so it raises no update event; a consumer of
the enriched value therefore has no moment to bind on the ordinary axes, and one bound to the create
event races the enriching listener.

> **Normative.** A phase name MUST be a lower-camel identifier, MUST be unique within its entity, and
> MUST NOT be a channel the implementation itself publishes.

> **Normative.** For each declared phase, the generated data-access layer of the entity MUST expose an
> operation that applies a set of column values and announces that phase as one write, such that no
> consumer can observe the announcement without the values.

### The event axis: onPhase

A glue consumer that binds an entity event MAY bind a declared phase:

```yaml
postings:
  - name: cogsPosting
    event: { onPhase: StockMovement, phase: costed }
```

> **Normative.** An `onPhase` binding MUST carry a sibling `phase:` naming a declared phase of the
> bound entity. A binding naming an undeclared phase MUST be rejected. A `phase:` key on a binding of
> another axis MUST be rejected.

> **Normative.** The `when:` guard is OPTIONAL on `onPhase`.

> **Normative.** A consumer bound to a phase MUST receive the record with the same payload, recipient
> paths, placeholders and guard semantics as a consumer bound to any other entity event.

When the bound entity is owned by another model, its phases are declared there and the name is not
resolvable from the consumer's model.

## DSL index

| Construct | What it does |
| --- | --- |
| `phases` (entity) | names the enrichment moments the entity announces, so a silent write has a channel |
| `event: { onPhase, phase }` | binds a consumer to a declared enrichment moment instead of the insert |
