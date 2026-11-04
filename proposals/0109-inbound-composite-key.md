# inbound: a lookup by a composite business key - by: [customer.code, plan.code], from: [customerCode, planCode]

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/135, https://github.com/eclipse-dirigible/dirigible/issues/8033
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8035

## The problem

An arrival lookup resolves **one** business key: in 1.11, `by` "MUST name a field of the target that
identifies at most one record". Many records are identified only by a **combination**: a
subscription by its customer and its plan, a price by its product and its currency. The sender knows
the combination's parts, not the record.

So a model that receives such a message has to store the parts instead of the record. It gives the
arriving entity its own relations to each part, each a single-key lookup, and resolves the record it
means later, by hand. The arriving record then carries a copy of a pair the record it means already
holds, and every reader has to resolve it again.

## The proposed shape

```yaml
entities:
  - name: Subscription
    unique:
      - fields: [customer, plan]
        message: "The customer already subscribes to this plan"
    relations:
      - { name: customer, kind: manyToOne, to: Customer, required: true }   # Customer.code is unique
      - { name: plan,     kind: manyToOne, to: Plan,     required: true }   # Plan.code is unique
  - name: UsageReport
    relations:
      - { name: subscription, kind: manyToOne, to: Subscription, required: true }
inbound:
  - name: usage
    source: { queue: usage.reported }
    create: UsageReport
    map:
      units: units
      subscription: { lookup: Subscription, by: [customer.code, plan.code], from: [customerCode, planCode] }
```

## Expected behaviour

A lookup's `by` and `from` may be **lists of the same length**, paired by position, naming a
**composite business key** of the target. Each `by` element is a field of the target, or a one-hop
`relation.field`: a to-one relation of the target and a field of its target that identifies at most
one record. The lookup fills the relation with the one target record whose key parts equal the
envelope's values, each `relation.field` part compared through the record its value identifies.

A miss anywhere - an envelope key absent, a part that identifies no record, no target record with
that combination - is the lookup's miss, and the arrival is rejected as for a single key. The
diagnostic reports every value of the combination. A lookup's `onMissing` (proposal 0108) governs
the whole composite lookup.

## Edge rules

- The target properties the elements name - each field itself, or the relation of a
  `relation.field` - MUST together be exactly the `fields` of one composite `unique` of the target,
  in any order. Anything else MUST be rejected: it could match several records.
- A `relation.field` element follows the single-key rule on its hop: the field MUST identify at most
  one record of the relation's target - declared unique, or that target's key - and be text or an
  integer, and not a translated property. A hop across models, and a hop of more than one relation,
  MUST be rejected.
- `by` and `from` MUST be of the same length, at least two; a list of one MUST be written as the
  scalar form. The same property named twice, and a `from` element `"."`, MUST be rejected.
- The scalar form is unchanged. A composite lookup is valid wherever a lookup is: an arrival's `map`
  and a collection's element map, on every source.

## Prior art / workarounds

Single-key lookups to each part, and a hand-written resolution of the record they identify.
Reference implementation: https://github.com/eclipse-dirigible/dirigible/pull/8035.

## Specification text

**Anchor:** Declarative glue > inbound - arrivals from outside > Reading an arrival as an envelope -
`accept` / `map`, the normative block, after "A generator MUST refuse a `by` that does not, rather
than resolve a lookup that could match several records."

Add to the same normative block:

> `by` and `from` MAY instead be lists of the same length, at least two, paired by position: a
> **composite lookup**. Each element of `by` names a field of the target, or a one-hop
> `relation.field` - a to-one relation of the target, declared in the same model, and a field of
> that relation's target held to the rule above. The target properties the elements name - the
> field, or the relation of a `relation.field` - MUST be exactly the fields of one composite
> [`unique`](#unique--a-business-key-over-more-than-one-field) declared on the target, in any order;
> a generator MUST refuse any other combination. A list of one, lists of unequal length, a property
> named twice, a `from` element `"."` and a hop of more than one relation MUST be rejected.
>
> The lookup matches the one target record whose every part equals the envelope's value, a
> `relation.field` part through the record its value identifies. An absent envelope key, a part that
> identifies no record, and a combination no target record carries are each a lookup matching none,
> reported with every value of the combination. A composite lookup is valid wherever a lookup is, and
> everything said of a lookup's miss applies to it as a whole.

## DSL index

No new row; the `inbound.accept` / `inbound.map` row gains "resolve a business key, or a composite
one declared `unique` on the target, to a relation".
