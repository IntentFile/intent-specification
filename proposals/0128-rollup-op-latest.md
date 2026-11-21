# A roll-up keeps a parent field equal to its newest child's value - `op: latest`

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/219
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/6350, https://github.com/eclipse-dirigible/dirigible/pull/8445

## The problem

1.11's roll-ups know a count (the default) and `op: sum`. A recurring shape is neither: a parent
whose headline value is the value of its **newest** child row - a currency's current rate from its
rate history, a contract's current price from its price rows. Without a construct for it the
parent's field is dead (nothing maintains it), and the newest value is a hand-written listener or a
report instead of a field that a list column, a check or a schedule's `where` can read.

## The proposed shape

```yaml
rollups:
  - { name: currentRate, entity: CurrencyRate, via: currency, field: rate,
      op: latest, of: rate, by: validFrom }      # the rate of the row with the greatest validFrom
```

## Expected behaviour

- `op: latest` keeps the parent's `field` equal to the `of` value of the child row with the
  greatest `by` date or timestamp.
- It recomputes on the child's create, update, delete and re-parenting, as every roll-up does: any
  of them can change which row is the newest, or its value.
- An empty child set writes no value onto the parent's `field`.
- The recompute writes only when the value changes, as for any roll-up.
- The parent's `field` is a roll-up target and so read-only by construction (0115): a full-row
  write of the parent keeps the computed value.

## Edge rules

- `op: latest` requires `of` and `by`. `by` MUST name a `date` or `timestamp` field of the child,
  and the parent's `field` MUST have the same type as `of`.
- `capacity`, `balance` and the status keys (`status`, `statusWhenFull`, `statusWhenPartial`) are
  refused with `op: latest`.
- Which of two children sharing the greatest `by` is copied is not specified.

## Prior art / workarounds

A hand-written listener on the child, or a report in place of the field. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/6350; the target's read-only rule is
https://github.com/eclipse-dirigible/dirigible/pull/8445.

## Specification text

**Anchor:** Declarative glue > rollups - denormalised parent totals, after the paragraph "A count
roll-up keeps a counter on a parent ... recomputation stops when values stop changing."

With `op: latest` the roll-up keeps `field` equal to the `of` value of the child with the greatest
`by` - the parent's current value read off its history, such as a currency's rate from its dated
rate rows:

```yaml
rollups:
  - { name: currentRate, entity: CurrencyRate, via: currency, field: rate,
      op: latest, of: rate, by: validFrom }
```

> **Normative.**
> A roll-up declaring `op: latest` MUST keep the parent's `field` equal to the `of` value of the
> child row with the greatest `by`, recomputed on the child's create, update, delete and
> re-parenting, and MUST leave `field` without a value when the parent has no children. `of` and
> `by` are REQUIRED with `op: latest`; `by` MUST name a `date` or `timestamp` field of the child,
> and `field` MUST have the type of `of`. `capacity`, `balance`, `status`, `statusWhenFull` and
> `statusWhenPartial` together with `op: latest` MUST be reported as authoring errors. Which of
> several children sharing the greatest `by` is copied is not specified.

## DSL index

The existing row gains `latest`:

| Construct | What it does |
| --- | --- |
| [`rollups`](#rollups--denormalised-parent-totals) | counts, sums, the newest child's value (`op: latest`), balance + status maintenance (a status set is also relinquished), onto a local or cross-model parent |
