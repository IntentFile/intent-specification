# A status guard and a capacity check hold under concurrency

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7827
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8813

## The problem

Three constructs guard a write by what a row holds:

- a create-from's `fromStatus:` (or its implied `sourceStatus` guard) - the source must stand in a
  listed status;
- a transition's `from:` - the record must stand in an allowed source status;
- a roll-up's `capacity` check - the children's sum must stay within the parent's capacity.

The text says each is refused, but not that the read and the write belong together. An
implementation that reads the row, checks, and then writes in a separate step lets concurrent
requests all read the same state and all pass: four clicks on one CONFIRMED proforma create four
invoices; one transition fired twice at once moves the record twice; two payments that each fit on
their own pay an invoice of 120 to 240.

## The proposed shape

No new key.

## Expected behaviour

- The guard and the write it protects are one unit: concurrent requests take turns, and each later
  one is judged against what the earlier one committed.
- A request refused by the guard writes nothing, and is answered with the refusal the guard already
  defines.

## Edge rules

- A capacity check gated by `guardAt:` - evaluated when a child is moved into a status - is not
  covered: an implementation is not required to serialise two children moved into the gate status
  at once.
- The soft `duplicate` warning is not a guard of this kind: a create has no existing row to hold.

## Prior art / workarounds

None within the model.

## Specification text

**Anchor:** three places, one sentence each, appended to the normative block named.

1. Declarative glue > `generates — create-from`, the normative block on `fromStatus`:

> The status guard MUST be decided against the source as it stands when the target is created: two
> concurrent invocations on one source MUST NOT both pass a guard that only one of them could pass
> alone.

2. Declarative glue > `transitions — guarded status flips`, appended to the opening paragraph as a
   normative block:

> **Normative.** A transition's `from:` and `when:` guards and the status write MUST be one unit: two
> concurrent invocations on one record MUST NOT both pass a guard that only one of them could pass
> alone, and a refused invocation MUST write nothing.

3. Declarative glue > `rollups — denormalised parent totals`, the normative block on `where:` /
`guardAt:`:

> A capacity check not gated by `guardAt:` MUST be decided against the children committed together
> with the write it judges: two concurrent child writes that each fit alone but not together MUST NOT
> both commit. An implementation is not required to serialise a check gated by `guardAt:`.

## DSL index

No new row.
