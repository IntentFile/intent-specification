# Mapping on arrival — a business key becomes a relation, and a gate ignores what is not understood

- **Status:** draft
- **Issue:** [IntentFile/intent-specification#30](https://github.com/IntentFile/intent-specification/issues/30)
- **Implementation:** [eclipse-dirigible/dirigible#6769](https://github.com/eclipse-dirigible/dirigible/issues/6769)
  ([PR #6839](https://github.com/eclipse-dirigible/dirigible/pull/6839))
- **Companion:** [`0017-declared-payload.md`](0017-declared-payload.md) — the same problem in the
  other direction, whose value vocabulary this one mirrors; and
  [`0018-outbound-departures.md`](0018-outbound-departures.md), whose own issue named this half.

## The problem

An `inbound` entry ingests the arriving JSON **as the entity**. That is only expressible when the
sender's payload already *is* the entity, field for field — and a real arrival contract is not. It is
an envelope, frozen between two products long before either one modelled anything:

```json
{ "messageId": "9f9d1c9e-...", "type": "user.assignment.requested", "version": 1,
  "tenantId": "acme", "appId": "library", "email": "new.user@example.com", "role": "User" }
```

Ingesting that needs three things the construct cannot say:

1. **Field mapping** — the envelope's keys are not the entity's names.
2. **A business key resolved to a relation** — `tenantId: "acme"` and `role: "User"` identify the
   `Tenant` and the `AssignmentRole` records, and what the entity stores are references to them. This
   is the single most common requirement of any arrival, and on its own it is enough to force a
   hand-written consumer.
3. **A type and version gate** — accept `user.assignment.requested` at version 1, and *ignore*
   anything else with a warning rather than failing it, so a sender rolling out version 2 does not
   fill the receiver's error queue with messages it will keep sending.

The consequence is the one worth avoiding: an application can declare its entity, its deduplication,
its process trigger **and** its arrival, and still need custom code purely because a relation arrives
as a name instead of a reference. The construct looks like it covers the case, and the hand-off is
discovered when the first payload lands.

The asymmetry is also hard to defend on its own terms. The format admitted the declared envelope for
messages **leaving** ([`0017`](0017-declared-payload.md)) on the argument that real contracts are not
records; an arriving message is the same contract read in the other direction, and until now it had to
be a record.

## The proposed shape

Two optional keys on an `inbound` entry, valid on **every** arrival, because they describe the payload
rather than the transport it came over.

```yaml
inbound:
  - name: userAssignments
    source: { queue: "codbex.user-assignment-requests" }
    accept: { type: user.assignment.requested, version: 1 }   # anything else: warn and ignore
    create: TenantUserAssignment
    map:
      messageId: messageId                                     # entity field <- envelope key
      email:     email
      seats:     seatCount
      tenant:      { lookup: Tenant,         by: tenantId, from: tenantId }   # business key -> relation
      role:        { lookup: AssignmentRole, by: name,     from: role }
```

`map:` omitted keeps the current behaviour exactly — the payload is the record — so no existing file
changes meaning and the simple case stays one line.

## Expected behaviour

**The gate.** `accept:` names envelope keys and the value each must carry. A message matching all of
them is ingested. A message that does not match is **acknowledged and ignored**, with a warning
recording what arrived and what was expected. It is deliberately *not* failed: failing an arrival asks
the sender to deliver it again, and a message this receiver structurally does not understand will be
identical every time. Over a request/response arrival the answer says the same thing — the message was
received and not acted on — rather than reporting a fault the sender did not commit.

**The projection.** `map:` fills the named property of `create:` from the named envelope key. A key the
map does not mention is not part of the record; an envelope key that is absent leaves its property
unset, which the entity's own required-value rules then judge like any other missing value. The
authored order is irrelevant to the result.

**The lookup.** `{ lookup: <Entity>, by: <field>, from: <envelopeKey> }` reads the envelope key, finds
the `lookup` entity whose `by` field equals it, and stores a reference to that record in the mapped
relation. Exactly one match fills the relation. **No match rejects the whole arrival** — with a
diagnostic naming the value that could not be resolved — because a record stored with an unresolved
reference is a record nobody can trace back to the party that asked for it, and it is indistinguishable
from one that never named a party at all.

**The write path is unchanged.** Whatever is declared, the record is saved through the entity's
ordinary write path, so validations, translations and the create event behave exactly as for any other
write. Mapping changes what the record is built from, never how it is stored.

## Edge rules

- **`by:` must identify at most one record** — a field declared unique, or the entity's key. A lookup
  that could match several rows would have to pick one, and picking silently is a worse outcome than
  failing; so a non-unique `by:` is refused when the file is read, not when two rows first collide.
- **`by:` must be a field a business key can travel as** — text or an integer. A temporal key would
  make the comparison depend on how the sender formatted a date, which is not a key.
- **A lookup fills a to-one relation only.** On a plain field it is meaningless: there is nothing to
  resolve to.
- **`lookup:` must name the relation's own target.** Two ways of saying which entity is read could only
  drift apart.
- **A mapped key must be a field or a to-one relation of `create:`.** A key naming neither is an
  authoring error, reported with the nearest declared name (per
  [`0014`](0014-unknown-keys-are-errors.md)).
- **A lookup value's keys are closed** — `lookup`, `by`, `from`, all three required. An unrecognised
  one is an error.
- **The entity's key MUST NOT be mapped.** It is assigned when the record is created. An arrival that
  carries its own identifier gives it a field of its own, declared unique — which is also what makes a
  redelivery of the same message refuse itself.
- **A valueless key is an error, not an empty one.** `accept: { type: }` and `map: { email: }` are
  reported. Read as "no value", the first would gate on nothing — accepting every message — and the
  second would fill nothing, in both cases losing an authored promise silently.
- **`accept:` values are scalars.** A nested object or list is not a gate.
- **A plain envelope key on a to-one relation** copies the target's key as it arrives, which remains
  valid: a sender that already speaks in references needs no lookup.
- **Same-model targets in this revision.** `lookup:` reads an entity declared in the same model.
  Resolving a business key against a model that only declares the entity by reference is a larger
  question and is left out deliberately rather than half-specified.

## Prior art / workarounds

A hand-written subscriber or endpoint alongside the generated application: parse the envelope, query
each register by its business key, build the entity, save it through the generated write path. It is
mechanical, it is written again in every application that consumes the same contract, and — because the
contract and the resolution live in that code — the arrival is invisible to anyone reading the model.

Some formats offer a general transformation language for this. That is the trade this proposal
deliberately does not make: three value forms and one lookup shape express the contracts people
actually have, while a transformation language moves the arrival out of the model and into a second
program written in a worse language. An arrival that needs more than this is an algorithm, and the
honest answer is a hand-written handler over the same write path.

## Specification text

**Anchor:** Declarative glue > `inbound — arrivals from outside` (immediately after that section's
normative block)

An arrival may declare how its payload is **read**. Without these keys the payload is the record; with
them it is an envelope, which is what an integration contract normally is.

```yaml
inbound:
  - name: userAssignments
    source: { queue: "user-assignment-requests" }
    accept: { type: user.assignment.requested, version: 1 }
    create: TenantUserAssignment
    map:
      messageId: messageId
      seats:     seatCount
      tenant:    { lookup: Tenant, by: tenantId, from: tenantId }
```

`accept` gates the arrival on the envelope keys it names. `map` fills a property of `create` from an
envelope key, or — given `{ lookup, by, from }` — from the record of another entity that a **business
key** identifies: the envelope carries a name, the entity stores a reference.

> **Normative.**
> `accept` and `map` are independent, both optional, and both valid on every arrival — they describe
> the payload, not the transport. An arrival declaring neither behaves exactly as one specified before
> them.
>
> A message matching every key of `accept` is ingested. A message that does not match MUST be
> acknowledged and ignored, with a diagnostic; it MUST NOT be failed, since redelivery cannot change
> the outcome. `accept` values are scalars.
>
> Each key of `map` MUST name a field or a to-one relation of `create`; the entity's key MUST NOT be
> mapped, being assigned on creation. A value is an envelope key, or a lookup declaring exactly
> `lookup`, `by` and `from`. A lookup MUST fill a to-one relation, `lookup` MUST name that relation's
> target, and `by` MUST name a field of the target that identifies at most one record — a field
> declared unique, or the target's key — and one that a business key can travel as, namely text or an
> integer. A generator MUST refuse a `by` that does not, rather than resolve a lookup that could match
> several records.
>
> A lookup matching exactly one record fills the relation. A lookup matching none MUST reject the
> arrival, reporting the unresolved value; it MUST NOT store the record with the relation unset. A key
> declared with no value at all — in either block — MUST be reported as an authoring error rather than
> read as an empty one.
>
> A mapped record MUST be saved through the entity's ordinary write path, exactly as an unmapped one
> is: mapping changes what the record is built from, never how it is stored.

## DSL index

| Construct | What it gives you |
| --- | --- |
| [`inbound.accept` / `inbound.map`](#inbound--arrivals-from-outside) | read an arrival as an envelope: gate on its type and version, map its keys onto the record, resolve a business key to a relation |
