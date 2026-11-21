# A roll-up status with one seed, and a full status that never outlives its sum

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7941, https://github.com/eclipse-dirigible/dirigible/issues/8312
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8809, https://github.com/eclipse-dirigible/dirigible/pull/8818

## The problem

A sum roll-up's `status:` moves the parent to `statusWhenFull` when the sum reaches the capacity and
to `statusWhenPartial` while it is positive and below it. Two shapes are left undefined.

**One seed.** A fixed asset is FULLY_DEPRECIATED once its depreciation reaches its cost, and has
nothing to say while it is being depreciated - it stays ACTIVE, an ordinary status of its own
lifecycle. The natural declaration names only `statusWhenFull`. The format does not say whether that
is valid, nor which band such a roll-up owns, nor what happens when the sum leaves the band (the last
depreciation reversed). A declaration naming neither seed has nothing to set at all.

**No displaced status at zero.** A status the roll-up sets, it relinquishes: the first move into a
roll-up-owned status records the status it displaced, and a sum back at zero restores it. But a
parent that already held `statusWhenPartial` when the sum first moved - the asset whose initial
ACTIVE is the partial seed - displaced nothing, so nothing is recorded. When the sum later returns to
zero from FULL, the rule "a roll-up-owned status with no recorded predecessor is left unchanged"
keeps the asset FULLY_DEPRECIATED with its whole cost left to depreciate.

## The proposed shape

```yaml
rollups:
  # one seed: the roll-up owns only the full band
  - { name: assetDepreciated, entity: Depreciation, via: Asset, field: depreciated,
      op: sum, of: amount, capacity: cost, balance: bookValue,
      status: Status, statusWhenFull: FULLY_DEPRECIATED }
  # both seeds, the partial one being the parent's initial status
  - { name: assetDepreciatedBoth, entity: Depreciation, via: Asset, field: depreciated,
      op: sum, of: amount, capacity: cost,
      status: Status, statusWhenFull: FULLY_DEPRECIATED, statusWhenPartial: ACTIVE }
```

## Expected behaviour

- A `status:` naming only `statusWhenFull` owns the full band (a positive sum at or above the
  capacity); one naming only `statusWhenPartial` owns the partial band (a positive sum below the
  capacity). A sum outside the named band relinquishes the status exactly as a zero sum does: the
  displaced status is restored and the record cleared.
- A `status:` naming neither seed is an authoring error.
- With both seeds declared, a zero sum on a parent holding `statusWhenFull` with no recorded
  predecessor moves it to `statusWhenPartial`. A parent resting in `statusWhenPartial` with no
  recorded predecessor is left alone, without a report. A recorded displaced status still wins.
- A one-seed roll-up has no status of its own to fall back to: a roll-up-owned status with no
  recorded predecessor is still left unchanged and reported.
- A roll-up declaring both seeds behaves exactly as before everywhere else.

## Edge rules

- The rule covers every path that changes the sum: the child's create, update, delete and
  re-parenting.
- There is still no `statusWhenEmpty`.

## Prior art / workarounds

A second, hand-written listener on the child that moves the status back; or a dummy partial seed
that the author does not want.

## Specification text

**Anchor:** Declarative glue > `rollups — denormalised parent totals`, the normative block on
relinquishing a status (the one ending "the key MUST be rejected as unknown"). Two changes.

1. Replace the sentence of that block beginning "A roll-up-owned status with no recorded
   predecessor" with:

> A roll-up-owned status with no recorded predecessor is not always a gap. Where the roll-up declares
> both seeds, a parent holding `statusWhenFull` when the sum returns to zero MUST be moved to
> `statusWhenPartial`, and a parent holding `statusWhenPartial` MUST be left unchanged without a
> report: a parent that held `statusWhenPartial` when the sum first moved displaced nothing. Where the
> roll-up declares one seed, a roll-up-owned status with no recorded predecessor - a parent set to
> the seed by hand, or one that reached it before the implementation kept the record - MUST be left
> unchanged and reported, not replaced by a guess.

2. Append after that block:

A roll-up may name one seed. Naming only `statusWhenFull` makes it own the full band, naming only
`statusWhenPartial` the partial one; a sum outside the named band gives the status back as a zero
sum does.

```yaml
rollups:
  - { name: assetDepreciated, entity: Depreciation, via: Asset, field: depreciated,
      op: sum, of: amount, capacity: cost, balance: bookValue,
      status: Status, statusWhenFull: FULLY_DEPRECIATED }
```

> **Normative.**
> A roll-up declaring `status:` MUST declare `statusWhenFull`, `statusWhenPartial` or both; naming
> neither MUST be reported as an authoring error. A roll-up naming only `statusWhenFull` MUST move the
> parent to it while the sum is positive and at or above the capacity, and one naming only
> `statusWhenPartial` while the sum is positive and below it. A sum outside the named band MUST
> relinquish the status as a zero sum does, on every path that changes the sum. A roll-up declaring
> both seeds MUST behave as before.

## DSL index

No new row; the `rollups` row is unchanged.
