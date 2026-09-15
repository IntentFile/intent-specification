# A create-from is guarded by the source's status

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7068,
  https://github.com/eclipse-dirigible/dirigible/issues/7150 (the affordance and the endpoint decide
  by one rule)
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7080,
  https://github.com/eclipse-dirigible/dirigible/pull/7201

## The problem

A `generates` create-from is offered as a button on the source record and served by an endpoint behind it. Nothing about it is conditional: it runs whenever it is invoked, as often as it is invoked.

That is fine for an action a user may legitimately repeat. It is wrong for the flow the construct exists for — minting a follow-up **document** from a source that can only produce one:

```yaml
generates:
  - name: invoice-from-proforma
    from: ProformaInvoice
    to: SalesInvoice
    forEntity: ProformaInvoice
    map: { Customer: Customer, ProformaInvoice: id }
    sourceStatus: INVOICED       # the proforma is INVOICED once the invoice exists
```

Pressing the button issues the invoice and flips the proforma to INVOICED. The button then **stays on the flipped record**, and the endpoint answers the next call exactly as it answered the first: a second invoice, for the same proforma, to the same customer. A user who double-clicks, or who comes back the next day unsure whether the run went through, double-invoices.

`sourceStatus` already declares what "already done" looks like for this action. Nothing consults it — which is why the gap survives review: both halves of the model read correctly, and the second document is a legitimate-looking record with its own number.

The only conditional trigger a create-from has is `event`, and it guards a different thing: an event-driven create-from carries an at-most-once guard over the back-reference and qualifies its moment with `event.when`. A create-from that is *clicked* has no such guard, and a document-issuing action is precisely the one where a person is the trigger.

An adjacent construct already has the answer. A `transitions` entry names the statuses its record may move **from**, and is refused from any other. A create-from cannot spell it `from`, because `from` there already names the source **entity**.

## The proposed shape

A create-from may declare the source statuses it may run from:

```yaml
generates:
  - name: invoice-from-proforma
    from: ProformaInvoice
    to: SalesInvoice
    forEntity: ProformaInvoice
    fromStatus: [CONFIRMED]      # the statuses the source may stand in
    sourceStatus: INVOICED
    map: { Customer: Customer, ProformaInvoice: id }
```

Statuses are seeded names or ids, as everywhere a status is named.

A create-from that declares `sourceStatus` and no `fromStatus` is guarded **implicitly** against exactly that status: a source already standing where the completion hook put it has been generated from. That is the minimal refusal, it is derived from what the author already wrote, and it means a model carrying this defect today is corrected without an authoring change.

## Expected behaviour

- The invocation is refused when the source does not stand in an accepted status, with an error naming the action and the source's current status. Nothing is created and the source is not modified.
- The refusal is decided **before** the target is created. A guard asked afterwards is not a guard.
- The affordance follows the guard: an action offered per record is not offered on a record whose status the guard would refuse. The refusal on the endpoint remains the contract — the affordance is not the enforcement. The two are one rule read twice: the affordance and the endpoint compare the same status value of the same record, by the same comparison a `transitions` guard uses, so a button is never live where the endpoint would refuse, whatever form the status value takes.
- A create-from that declares neither `fromStatus` nor `sourceStatus` is unguarded, exactly as before.

## Edge rules

- `fromStatus` guards the **invocation by a user**. An `event`-driven create-from has its own at-most-once guard over the back-reference and qualifies its moment with `event.when`, so `fromStatus` on a create-from that contributes no button is an authoring error rather than a key that is accepted and ignored.
- `fromStatus` requires the source to declare an entity-status relation: without one there is no value to read.
- `fromStatus` requires a per-record action. An action scoped to the whole view has no record whose status could be read.
- `fromStatus` must not list the `sourceStatus` the action itself writes: the completion hook moves the source there once the target exists, so allowing it back is a second document from the same source.
- An explicit `fromStatus` replaces the implied guard rather than adding to it — the author has stated the whole rule.
- The guard reads the source's status; it says nothing about the target. A target that no longer counts (retired into a cancelled or void stage) is the concern of the event trigger's at-most-once guard, and of the declared return of the source to an earlier status.

## Prior art / workarounds

Today the two available answers are both outside the model. Either the button is left offered and the duplicate is caught by whoever reads the ledger — the outcome the implementation issue records, with two invoices already in the customer's hands — or the create-from is driven by an event instead of a click, which buys the at-most-once guard at the cost of taking the decision away from the person whose job it is to make it. A hand-written check in front of the endpoint is not available either: the endpoint is generated.

`transitions` has had the guard from the start, in the form this proposal follows.

## Specification text

**Anchor:** generates — create-from (after the `sourceStatus` paragraph)

A create-from may declare the source statuses it may run from:

```yaml
generates:
  - name: invoice-from-proforma
    from: ProformaInvoice
    to: SalesInvoice
    forEntity: ProformaInvoice
    fromStatus: [CONFIRMED]           # the statuses the source may stand in
    sourceStatus: INVOICED            # where the source lands once the invoice exists
    map: { Customer: Customer, ProformaInvoice: id }
```

> **Normative.**
> A create-from declaring `fromStatus` is refused when the source record does not stand in one of the listed statuses; the refusal names the action and the source's current status, and is decided before anything is created. A per-record action is not offered on a record its guard would refuse, and the refusal on the action's own endpoint remains the enforcement. A create-from that declares `sourceStatus` and no `fromStatus` is guarded against exactly that status, since a source standing where the completion hook put it has already been generated from; a create-from declaring neither is unguarded. `fromStatus` guards an invocation by a user: on a create-from that contributes no button it is an authoring error, as it is on an action scoped to the whole view, on a source declaring no entity-status relation, and when it lists the `sourceStatus` the action itself writes.

## DSL index

| Construct | What it does |
| --- | --- |
| [`generates[].fromStatus`](#generates--create-from) | the source statuses a create-from may run from — so an already-invoiced proforma stops offering, and stops accepting, a second invoice |
