# A status badge takes its colour from the seed row - `stage` colours it, an optional `variant` overrides it

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/154, https://github.com/eclipse-dirigible/dirigible/issues/8373
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8380

## The problem

A generated application shows a record's status as a coloured badge. With nothing in the model to go
on, the colour is guessed from keywords in the status name, so a nomenclature named in another
language, or with names the guess does not know, renders every status the same colour.

The seed row already says what a status **means**: `stage` classifies it as `draft`, `live`,
`cancelled` or `void` (1.11, Data, seeds & naming > stage), and that meaning is what the colour
should follow. 1.11 says `stage` is consumed by reports and the lifecycle; it says nothing about
how a status is rendered, and offers no way to give one status a colour of its own.

## The proposed shape

```yaml
seeds:
  - name: order-statuses
    entity: OrderStatus
    rows:
      - { id: 1, name: DRAFT,     stage: draft }                     # outlined
      - { id: 2, name: POSTED,    stage: live }                      # positive
      - { id: 3, name: CANCELLED, stage: cancelled, variant: warning } # the variant wins over the stage
```

## Expected behaviour

- A status badge is coloured by its seed row's `stage`: `draft` outlined, `live` positive,
  `cancelled` negative, `void` muted.
- `variant` is a closed vocabulary - `success | info | warning | destructive | neutral | muted` -
  and wins over the row's `stage`.
- A row with neither keeps the generator's own rendering of an unclassified status (the name-based
  guess); a nomenclature classified in neither way renders exactly as before.
- Like `stage`, `variant` is **metadata, not data**: it is never a column of the seeded table.

## Edge rules

- Refused when the intent is read: a value outside the vocabulary; a `variant` on a row without its
  id; a `variant` on a seed of an entity that is not the target of a `function: EntityStatus`
  relation in this model; a status nomenclature declaring its own `variant` property (the marker
  would collide).
- A status nomenclature owned by another model is seeded there, so no colour is resolved for it in
  the referencing model; its badge renders as an unclassified one.
- Every page that shows the status badge - lists, record pages, document pages, the personal and
  partner pages, a status stepper - follows the same colours.

## Prior art / workarounds

The keyword guess over the status name, which works only for names it knows. Reference
implementation: https://github.com/eclipse-dirigible/dirigible/pull/8380.

## Specification text

**Anchor:** Data, seeds & naming > stage - what a status means to the lifecycle: a sentence and a
`Colour` column in the stage table, and a new subsection after it, before Status references - name,
not number. **Amends** the row-key rule of Data, seeds & naming > seeds: "or the `stage` marker
below" becomes "or the `stage` and `variant` markers below".

The stage table gains a column:

| Stage | Meaning | Colour |
| --- | --- | --- |
| `draft` | Nobody has issued it yet - visible to its author, not yet economically real. | outlined |
| `live` | It counts: issued, sent, paid - anything in normal circulation. | positive |
| `cancelled` | Withdrawn before it ever became live. | negative |
| `void` | Deliberately retired while keeping its number - out of circulation by design. | muted |

Appended after the paragraph that follows the table:

A status's stage is also what its badge looks like: wherever a generated page shows the record's
status, the badge takes the colour of the row's stage.

New subsection:

#### variant - the badge colour

A row whose meaning is right but whose colour should differ - a cancellation that should read as a
warning rather than a failure - declares `variant`, a closed vocabulary that wins over the row's
stage:

```yaml
rows:
  - { id: 3, name: CANCELLED, stage: cancelled, variant: warning }
```

| Variant | Colour |
| --- | --- |
| `success` | positive - as `live` |
| `info` | informational |
| `warning` | cautionary |
| `destructive` | negative - as `cancelled` |
| `neutral` | outlined - as `draft` |
| `muted` | muted - as `void` |

A `variant` changes how the status looks, never what it means: reports, scopes and the lifecycle
read the stage alone. A row may carry a `variant` without a `stage`.

> **Normative.**
> A conforming generator MUST colour a status badge by its seed row: by the row's `variant` when it
> declares one, otherwise by its `stage` as the stage table gives it, on every generated page that
> shows the status. A row that declares neither MAY be rendered as the generator renders an
> unclassified status, and a nomenclature none of whose rows declares either MUST render as it did
> before. Like `stage`, `variant` is metadata, not data: it MUST NOT be emitted as a column of the
> seeded table. A value outside the vocabulary, a `variant` on a row that does not carry the entity's
> primary key, a `variant` on a seed of an entity that is not the target of a `function:
> EntityStatus` relation in this model, and a status nomenclature that declares its own `variant`
> property MUST be reported as authoring errors. A nomenclature owned by another model is seeded
> there, so no colour is resolved against it; its badge renders as an unclassified one.

## DSL index

The `stage` row becomes:

| Construct | What it gives you |
| --- | --- |
| [`stage`](#stage--what-a-status-means-to-the-lifecycle) | classify a status: draft / live / cancelled / void - and so its badge colour: outlined / positive / negative / muted |

New row:

| Construct | What it gives you |
| --- | --- |
| [`variant`](#variant---the-badge-colour) | a status seed row's badge colour, overriding its stage's: success / info / warning / destructive / neutral / muted; metadata, never a column |
