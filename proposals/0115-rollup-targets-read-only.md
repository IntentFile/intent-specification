# A roll-up's target is read-only by construction

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/163, https://github.com/eclipse-dirigible/dirigible/issues/8442
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8445

## The problem

1.11 calls a roll-up target derived state "which a user write already may not set" (the flow-owned
status section), but the rollups chapter never says so, and most models declare the target as a
plain numeric field without `readOnly`. A full-row update of the parent that leaves the total out
then writes it back empty, and one that carries another value overwrites the computed total; both
stay wrong until the next child event recomputes them. An invoice paid 50 reads an empty `paid`
after its customer address is corrected.

This is a clarification of meaning, not a new key.

## The proposed shape

No new key. The example that must hold, with neither target marked `readOnly`:

```yaml
entities:
  - name: SalesInvoice
    fields:
      - { name: total,   type: decimal }
      - { name: paid,    type: decimal }
      - { name: balance, type: decimal }
rollups:
  - { name: invoicePaid, entity: Allocation, via: SalesInvoice, field: paid,
      op: sum, of: amount, capacity: total, balance: balance }
```

An update of the invoice that omits `paid` and `balance`, or carries other values for them, keeps
the stored totals.

## Expected behaviour

The fields a roll-up writes - its `field`, whatever its `op`, and the `balance` of a capacity sum -
are read-only whether or not the author marks them:

- a create drops a value sent for the target;
- an update keeps the stored value; a differing value is ignored, not refused, as for any
  `readOnly` field, so an edit form that sends the stored value back keeps working;
- the generated forms render the target read-only;
- the roll-up itself still writes the target.

The target has no value until the first child arrives. An opening value - a balance that starts
equal to the capacity - is a `calculatedOnCreate` on the target, which is computed after the
create has dropped the sent value.

## Edge rules

- A target declared `required` with no server-side source (`defaultValue`, `calculatedOnCreate`) is
  refused at generation: the create drops its value, so every create of the parent would fail.
- A cross-model parent's target is not affected here: it belongs to the model that owns it.
- The status a capacity roll-up's `status:` moves is not a target in this sense and keeps its own
  protection: an omitted value keeps the stored one, and a changed one is refused as a conflict.

## Prior art / workarounds

`readOnly: true` on every target by hand, which most models and fixtures did not write.
Reference implementation: https://github.com/eclipse-dirigible/dirigible/pull/8445.

## Specification text

**Anchor:** Declarative glue > rollups - denormalised parent totals, after the paragraph "A
roll-up writes onto the child's own parent ... a concurrent edit to another column of the same row
is never reverted."

The fields a roll-up writes belong to it. Declaring a target `readOnly` is not needed and changes
nothing.

> **Normative.**
> The fields a roll-up writes - its `field`, whatever its `op`, and the `balance` of a capacity
> sum - MUST be read-only whether or not the author marks them. A create MUST drop a value sent for
> a target, an update MUST keep the stored value (a differing value is ignored, not refused), and a
> target has no value until the first child arrives. A target declared `required` with no
> server-side source (`defaultValue`, `calculatedOnCreate`) MUST be refused at generation. A
> cross-model parent's target belongs to the model that owns it.

## DSL index

No new row; the `rollups` row is unchanged.
