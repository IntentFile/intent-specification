# outbound — a record may leave on a queue or a topic

- **Status:** draft
- **Issue:** [IntentFile/intent-specification#30](https://github.com/IntentFile/intent-specification/issues/30)
- **Companion:** [`0017-declared-payload.md`](0017-declared-payload.md) — the envelope this construct
  sends; and [`0012-glue-event-axis.md`](0012-glue-event-axis.md) — the axis it binds to.

## The problem

The format can describe a record **arriving** on a queue or a topic and cannot describe one
**leaving**. `inbound:` gained arrival sources; `integrations:` — the only "tell another system"
construct — is HTTP by construction: a method and a URL. There is no transport axis, so an
application that raises a business event for another system falls out of the format entirely.

Two things make the asymmetry hard to defend.

**The mirror argument that admitted arrivals applies unchanged.** An arrival was accepted as *"a
transport, not a second data path"* — the same `create:` through the same repository, differing only
in where the record showed up. A departure is the same one-line call-out shape as the HTTP POST
already specified, aimed at a different address.

**A conforming application is already an event publisher.** Implementations of this format publish
every write of a generated entity on an internal channel, and their whole reaction layer is
subscriptions to those channels — that is how a notification, a roll-up and a process trigger are all
built. The behaviour exists and is exercised on every write; only the vocabulary to point one of
those events *outward*, under a name someone else agreed to, is missing.

A concrete case. Two products agree that raising a membership suspends the member's borrowing rights
in the other one. The publishing application can model the entity, its lifecycle, its numbering, its
process — and then needs a hand-written subscriber on its own internal channel that calls the
platform's producer, because the model has no way to say "this event also leaves". The contract then
lives in that code, where nobody reading the model can see it.

## The proposed shape

A departure block mirroring `inbound:`, with exactly one channel per entry.

```yaml
outbound:
  # the record's own representation, on a queue - one consumer takes each message
  - name: publishOrder
    event: { onCreate: Order }
    to: { queue: "orders.outbound" }

  # a declared envelope, on a topic - every subscriber receives it
  - name: announceActivation
    event: { onStepCompleted: { process: OrderApproval, step: activate }, when: "channel != internal" }
    to: { topic: "order-activations" }
    payload:
      type: "order.activated"
      version: 1
      messageId: "{uuid}"
      tenantId: "{tenant}"
      reference: number
      customer: customer.name
```

## Expected behaviour

`to:` names **exactly one** of `queue` / `topic`. Two channels are two departures wearing one name;
none is a promise with nowhere to land. Both MUST be rejected as authoring errors, mirroring the
arrival rule.

The entry binds to the **same event axis** every reacting block uses — entity lifecycle events and
process-step events — and takes the same `when:` guard, so no action needs to know which axis fired
it. With no `payload:` the body is the record's own representation, exactly what an HTTP integration
forwards today; with one, it is the declared envelope of proposal 0017, resolved by the same rules.

**Delivery semantics, stated rather than implied.** The departure is published **after** the write it
reacts to is persisted, and is **not** transactional with it: a failure MUST be recorded and MUST NOT
fail the write, matching the rule the notify block already sets. Ordering, exactly-once delivery and
an outbox are explicitly **not** promised, and a conforming implementation MUST say so in its
documentation rather than leave an author to assume otherwise — an author who assumes an outbox
writes a different application than one who knows there is none.

Conversation-shaped transports — acknowledgement protocols, request-reply correlation, backoff policy
— stay beyond the scope boundary, as they are today (see proposal 0003).

## Edge rules

- **Naming a channel is not addressing a deployment.** A destination name in the model is a name the
  *application* owns; whether two separate deployments sharing one broker can meet on it is a
  property of the implementation's isolation, not of the format. An implementation that renames
  destinations per tenant MUST document how an author declares a destination that is a **contract
  with someone else**, because without that a departure is unreachable from outside the deployment
  and the construct silently means less than it reads.
- **The declaration is the contract.** An implementation MUST NOT widen a declared payload with
  fields the author did not name (no "and also the whole record, for convenience"), because the
  point of declaring the envelope is that adding a column does not change what leaves.
- **A departure is not an ordering mechanism.** Two departures on the same event have no defined
  relative order, and neither do two events on the same record.
- **An unresolvable payload value fails the departure, loudly, at generation** — emitting the record
  instead would put a different contract on the wire under the same name.

## An alternative shape considered

The transport could instead be a new key on `integrations:` (`to:` replacing `url:`), avoiding a
second construct that means "tell someone". That reads worse in one specific way: `integrations:` is
*call another system's API and it answers*, while this is *emit an event and forget*, and their
failure semantics differ accordingly — a failed call is a failed call, a failed announcement is a
missed announcement. Naming the departure `outbound:` also pairs it with `inbound:`, which is where
an author will look for it. Both shapes are otherwise identical, and proposal 0017 is unaffected by
the choice.

## Prior art / workarounds

Hand-written publisher code alongside the generated application: a subscriber on the entity's own
internal event channel that assembles the message and calls the platform's producer. It is
mechanical, repeated in every application that participates in the same integration, and — because
the contract lives in that code — invisible to anyone reading the model. The nominal hand-off for
this class of requirement (an integration route in the platform's integration technology, per
proposal 0003) does not necessarily cover it either: an implementation's route engine may carry no
messaging component at all, in which case the "designated hand-off" is a third hand-written thing.

The **arrival-mapping** half of issue #30 — `accept:` gating and a `map:` that resolves a business
key arriving as a name onto a stored relation — is deliberately left out of this proposal. It is a
change to `inbound:`, not to the departure, and it should stand or fall on its own.
