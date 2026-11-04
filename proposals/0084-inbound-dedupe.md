# inbound: dedupe - a repeated identifier is acknowledged and skipped instead of dead-lettered

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/145, https://github.com/eclipse-dirigible/dirigible/issues/6852
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8088

## The problem

On an at-least-once source a redelivered message is routine, but in 1.11 an `inbound` arrival's only
answer to it is the identifier's `unique: true` constraint. The refused save fails the message, the
channel redelivers it a few times, and it ends among the undeliverable messages - a **correct**
outcome filed next to the misses a lookup's `onMissing: deadLetter` (proposal 0108) sends there on
purpose, while a "source queue drained" check reads green because the message moved.

1.11 says a message that does not match `accept` "MUST be acknowledged and ignored, with a
diagnostic; it MUST NOT be failed, since redelivery cannot change the outcome". A message that was
already ingested is in the same position - redelivering it cannot change the outcome either - and
the format says nothing about it.

## The proposed shape

```yaml
inbound:
  - name: orders
    source: { queue: "shop.orders" }
    accept: { type: order.placed, version: 1 }
    create: Order
    dedupe: orderId                       # a seen identifier is acknowledged and skipped
    map:
      orderId: orderId
      customer: { lookup: Customer, by: code, from: customerCode, onMissing: deadLetter }
```

## Expected behaviour

After the `accept` gate and **before** any lookup, the arrival reads whether a record of `create`
already holds the mapped value of `dedupe`, converted exactly as `map` converts it. If one does, the
message is acknowledged and ignored with a diagnostic naming the arrival, the field and the value:
nothing is saved, no create event fires, and no lookup runs - so a repeat whose lookup would miss
**now** is still recognised as a repeat, and never reaches the undeliverable messages.

- A **queue** or **topic** arrival acknowledges the repeat.
- A **path** arrival answers the repeat as it answers an `accept` miss: 202, with a body saying the
  message was ignored because its identifier is already ingested.
- A **folder** arrival skips the repeated record; the file is still moved to the processed files.

## Edge rules

- `dedupe` MUST name a field of `create` that `map` fills from an envelope key, declared
  `unique: true` **and** `required: true` - an empty value is distinct from every other in a unique
  constraint, so an optional key would dedupe nothing. Each MUST be rejected otherwise.
- The race is unchanged: two deliveries that both pass the read are still refused by the unique
  constraint, as before.
- Out of scope: a composite identity, and updating the stored record on a repeat (an upsert).
- An arrival without `dedupe` behaves, and generates, exactly as in 1.11.

## Prior art / workarounds

The unique constraint alone, with the repeats filed among the undeliverable messages; or a
hand-written consumer that looks the identifier up first. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8088.

## Specification text

**Anchor:** Declarative glue > inbound - arrivals from outside > Reading an arrival as an envelope -
`accept` / `map`, a new paragraph and normative block after the normative paragraph on `accept`
("A message matching every key of `accept` is ingested. ...").

On an at-least-once channel a message arrives more than once. An arrival may name the field that
identifies it, so that a repeat is recognised as one rather than refused by the store:

```yaml
inbound:
  - name: orders
    source: { queue: "shop.orders" }
    accept: { type: order.placed, version: 1 }
    create: Order
    dedupe: orderId
    map:
      orderId: orderId
      customer: { lookup: Customer, by: code, from: customerCode }
```

> **Normative.**
> `dedupe` MUST name a field of `create` that `map` fills from an envelope key, and that field MUST
> be declared `unique: true` and `required: true`; a generator MUST reject a `dedupe` that does not.
>
> After the `accept` gate and before any lookup is resolved, an arrival declaring `dedupe` MUST
> determine whether a record of `create` already holds the envelope's value for that field, converted
> as `map` converts it. When one does, the message is a **repeat**: it MUST be acknowledged and
> ignored, with a diagnostic naming the arrival, the field and the value, exactly as a message that
> does not match `accept` is, and it MUST NOT be failed. Nothing of it is stored, no create event is
> raised and no lookup is resolved. A `path` arrival MUST answer a repeat as it answers an `accept`
> miss (202); a `folder` arrival MUST skip the repeated record and move the file as it moves any
> other read file.
>
> The check does not replace the field's unique constraint: two deliveries that both pass it are
> still refused by the store, as without `dedupe`. An arrival declaring no `dedupe` behaves exactly
> as one specified before it.

## DSL index

| Construct | What it gives you |
| --- | --- |
| [`inbound.dedupe`](#reading-an-arrival-as-an-envelope--accept--map) | the unique, required field that identifies an arrival: a message whose value is already stored is acknowledged and skipped, before any lookup, instead of being refused by the store and dead-lettered |
