# The click endpoints and the print feed demand the record's gates

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/163, https://github.com/eclipse-dirigible/dirigible/issues/7913, https://github.com/eclipse-dirigible/dirigible/issues/7921
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7922, https://github.com/eclipse-dirigible/dirigible/pull/8418

## The problem

A create-from, a transition and a print each act on a record through an endpoint of their own,
beside the record's own create, read, update and delete. 1.11 gates the record through
`permissions`, but says nothing about these three endpoints, and they were generated ungated: any
signed-in user, one holding no role at all, could create the target document of a create-from and
flip its source's status, move a record through a transition, and read any printable document -
while the entities' own endpoints refused the same user with 403. A print feed for an id that does
not exist answered with an empty document rather than a 404.

This is a clarification of what a conforming generator must refuse, not a new key.

## The proposed shape

No new key. The gates are the ones the record's own endpoints already demand:

```yaml
permissions:
  - { role: Sales,  can: [Quote:write, SalesOrder:write] }
  - { role: Viewer, can: [Quote:read] }
generates:
  - { name: order-from-quote, from: Quote, to: SalesOrder, sourceStatus: ORDERED }
transitions:
  - { name: CancelQuote, forEntity: Quote, from: [SENT], setStatus: CANCELLED }
```

`Sales` may run both clicks; a user who may write only `Quote` may run `CancelQuote` but not
`order-from-quote`; `Viewer` may print a quote and may run neither click; a user with no role is
refused all three.

## Expected behaviour

- **Create-from.** It demands a write grant on its source **and** on its target: it creates the one
  and, with `sourceStatus`, writes the other.
- **Transition.** It demands a write grant on the record it moves.
- **Print feed.** It demands a read or a write grant on its root document, and only on the root:
  the nodes it prints - related records, the composition items - are not gated one by one. A user
  who may read an invoice prints it, customer included.
- The refusal is a 403, decided before any record is loaded, so a refused caller cannot probe which
  ids exist: a real id and a missing one are refused alike. A print feed for a missing root,
  requested by a caller who holds the gate, is a 404.
- A grant is the one the entity's own endpoints demand: the roles a `permissions` entry grants on
  it, or the generator's default role for the entity where no entry names it.

## Edge rules

- An end owned by another model is gated by what its owning model declares.
- A model generated before these gates fails closed: an endpoint whose gate the generator did not
  record admits nobody until the model is regenerated, rather than staying ungated.
- A render the application makes itself - an attached print of a notify block, a snapshot, a
  scheduled message - is not a caller's request and is not gated; it often runs with nobody signed
  in.
- Whether a button is offered to a user its endpoint refuses is not decided here; the endpoint's
  refusal is the enforcement.

## Prior art / workarounds

None in the model: the endpoints were reachable by any signed-in user. Reference implementations:
https://github.com/eclipse-dirigible/dirigible/pull/7922 and
https://github.com/eclipse-dirigible/dirigible/pull/8418.

## Specification text

**Anchor:** Scoped surfaces & roles > permissions, appended. Amends the paragraph "Generates a
deduplicated set of roles ..." in that the grants also gate the create-from, transition and print
endpoints the model generates.

The grants that gate an entity's own endpoints gate the endpoints that act on its records as well:
a create-from, a transition and a print feed.

> **Normative.**
> A create-from MUST demand a write grant on its source AND on its target; a transition MUST demand
> a write grant on the record it moves; a print feed MUST demand a read (or write) grant on its root
> document, and only on the root - the nodes it prints are not gated one by one. The refusal is a
> 403 decided before any record is loaded, so a refused caller cannot probe which ids exist; a print
> feed for a missing root is a 404. A cross-model end's gate is the one its owning model declares.
> A model generated before these gates fails closed until it is regenerated.

## DSL index

No new row; the `permissions`, `generates` and `transitions` rows are unchanged.
