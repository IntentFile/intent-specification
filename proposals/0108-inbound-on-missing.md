# inbound: onMissing on a lookup - a message whose lookup resolves nothing is dead-lettered, not dropped

- **Status:** released in [1.12](../versions/1.12.md)
- **Issue:** https://github.com/IntentFile/intent-specification/issues/134, https://github.com/eclipse-dirigible/dirigible/issues/8031
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8034

## The problem

An arrival's `map` may fill a relation with a lookup, `{ lookup, by, from }`, and 1.11 says "a lookup
matching none MUST reject the arrival, reporting the unresolved value". It does not say what
rejecting means on a **messaging** channel. An implementation that acknowledges the message has lost
it: no record, no redelivery, nothing a person can find but a diagnostic line.

A lookup that matches nothing is usually a configuration error on one of the two sides - a sender
pointed at the wrong receiver, a register record not created yet. Unlike a message that fails
`accept`, which 1.11 says "MUST NOT be failed, since redelivery cannot change the outcome", a
redelivery **can** change a lookup's outcome: the record may exist by then. And a message the
channel dead-letters can be seen and replayed once the configuration is fixed.

## The proposed shape

```yaml
inbound:
  - name: orders
    source: { queue: orders.placed }
    accept: { type: order.placed, version: 1 }
    create: Order
    map:
      number:   number
      customer: { lookup: Customer, by: code, from: customerCode, onMissing: deadLetter }
```

## Expected behaviour

A lookup may declare **`onMissing`**, the disposal of a message when that lookup resolves nothing -
the envelope lacks the key it reads, or no single record matches:

- **`drop`** (the default): the arrival is rejected and the message is **acknowledged**. This is the
  behaviour of a lookup that declares no `onMissing`, so every existing model keeps its meaning.
- **`deadLetter`**: the arrival is rejected by **failing the delivery**. The channel's redelivery
  and dead-letter handling then apply: the message is retried, and once the retries are exhausted it
  is kept where the channel keeps undeliverable messages. A retry that finds the record ingests the
  message normally.

Either way nothing of the arrival is stored, and the diagnostic names the value that did not
resolve.

## Edge rules

- `onMissing` is valid on a lookup of an arrival with a `queue` or `topic` source, at the top level
  of `map` and in a collection's element map. On a `path` arrival it MUST be rejected: the caller is
  answered, so there is nothing to dead-letter. On a `folder` arrival it MUST also be rejected: a
  rejected file is already kept apart.
- Its value MUST be `drop` or `deadLetter`; any other value MUST be rejected.
- An element lookup that resolves nothing disposes of the whole arrival by its own `onMissing`.
  Every element is resolved before anything is stored, so nothing is half-written.
- The other rejections of an arrival - a payload that is not an envelope, a collection of the wrong
  shape, more than `max` elements - are not governed by `onMissing`.
- A model whose lookups declare no `onMissing` generates exactly as in 1.11.

## Prior art / workarounds

A hand-written consumer that fails the delivery instead of returning. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8034.

## Specification text

**Anchor:** Declarative glue > inbound - arrivals from outside > Reading an arrival as an envelope -
`accept` / `map`, the normative block; and An envelope array as composition children - collections,
its normative block.

**Amends** the sentence "A value is an envelope key, or a lookup declaring exactly `lookup`, `by` and
`from`." to read:

> A value is an envelope key, or a lookup declaring exactly `lookup`, `by` and `from`, and optionally
> `onMissing`.

After "A lookup matching none MUST reject the arrival, reporting the unresolved value; it MUST NOT
store the record with the relation unset.", add to the same normative block:

> A lookup MAY declare `onMissing`, the disposal of a message whose lookup resolves nothing - the
> envelope lacks the key it reads, or no single record matches. With `drop`, the default, the
> rejected message MUST be acknowledged. With `deadLetter`, the delivery MUST be failed, so the
> channel's redelivery and dead-letter handling apply; a redelivery that resolves the lookup MUST
> ingest the message as any other. Either way nothing of the arrival is stored. `onMissing` is valid
> only on an arrival with a `queue` or `topic` source - a `path` arrival answers its caller, and a
> `folder` arrival already keeps a rejected file apart - and its value MUST be `drop` or
> `deadLetter`; elsewhere, or with another value, it MUST be rejected.

In the collections block, after "or when an element lookup matches no record. A rejected arrival
stores nothing.", add:

> An element lookup MAY declare `onMissing` under the same rules; an element whose lookup resolves
> nothing disposes of the whole arrival by that lookup's `onMissing`. The other rejections listed
> here are not governed by `onMissing`.

## DSL index

| Construct | What it gives you |
| --- | --- |
| [`inbound.map` lookup `onMissing`](#reading-an-arrival-as-an-envelope--accept--map) | what a queue or topic arrival does with a message whose lookup resolves nothing: `drop` acknowledges it (the default), `deadLetter` fails the delivery so the channel retries it and keeps it as undeliverable |
