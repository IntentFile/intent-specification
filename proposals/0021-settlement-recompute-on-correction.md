# Settlement allocation recomputes when the payment changes

- **Status:** draft
- **Issue:** <!-- link the discussion issue, if any -->
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/issues/6818

## The problem

`settlements` allocates a payment across the payer's open invoices. Today the only moment that is
described is the payment's arrival, and every implementation has read it that way: the allocation
runs once, when the payment record is created.

Payments are not settled at the moment they are typed in. A payment booked for the wrong amount is
corrected the next day; a payment entered on Friday from a bank statement is completed on Monday
when the counter-party is identified; a payment captured as a draft is amended before anyone treats
it as money. In each case the record that finally carries the real amount is not the record that was
created, and the allocation - and therefore every invoice's settled figure - is left describing the
amount the payment used to have. Nothing reports this: the invoice simply shows a settled amount
that the payment no longer supports, and the difference surfaces months later in a receivables
reconciliation.

The same gap swallows the opposite correction. A payment reduced after it was allocated leaves
invoices carrying allocations larger than the payment they draw on, so the sum of the allocations no
longer equals the payment.

## The proposed shape

No new keys. The existing declaration gains a defined behaviour over the payment's whole life:

```yaml
settlements:
  - name: autoAllocate
    junction: SalesInvoiceCustomerPayment
    invoice: SalesInvoice
    payment: CustomerPayment
    amount: amount
    total: total
    paid: paid
    pot: amount
    order: date                       # allocate oldest first
    match: [Customer, Currency]
    status: Status
    payableStatuses: [3, 4, 6]
```

## Expected behaviour

Allocation runs on the payment's create **and** on every subsequent change to the payment, and each
run is a recompute of the payment's *unallocated* balance rather than an addition to what is already
allocated:

- the amount available to allocate is the payment's `pot` minus everything already allocated to it
  through the junction;
- when that balance is positive it is spread over the payer's open invoices exactly as on create;
- when it is negative - the payment now covers less than it is allocated to - the excess is given
  back, so the allocations again sum to the payment;
- when it is zero the run changes nothing.

A re-delivered, replayed or duplicated change therefore converges on the same allocation instead of
double-allocating the payment.

## Edge rules

- The excess is released from the most recently created allocations first, so the invoices that were
  settled earliest stay settled; the last allocation touched may be reduced rather than removed.
- Releasing is an ordinary write to the junction, so an invoice's `paid` roll-up follows it down the
  same way it followed the original allocation up.
- Only the payment's own changes re-run the allocation. An invoice becoming payable is the other,
  already-specified direction (the on-invoice pull).
- A payment whose `pot` is unset allocates nothing, as before.
- Allocation stays **eventually consistent, not transactionally exact** under concurrency, like every
  other recompute-on-event construct.

## Prior art / workarounds

Today an author who needs a correction to take effect deletes the allocation rows by hand and
re-saves the payment to make an implementation re-run the create path, or writes the whole
allocation by hand and abandons the construct. Neither is discoverable, and the first is only
attempted by someone who already knows the allocation is stale.

## Specification text

**Anchor:** Declarative glue > settlements — payment allocation

> **Normative.** A conforming generator MUST run the allocation on the payment's creation and on
> every subsequent change to the payment record. Each run MUST allocate the payment's unallocated
> balance - its `pot` less every amount already allocated to it through the `junction` - rather than
> allocate the `pot` again: a run with a positive balance spreads it over the payer's open invoices
> in `order`, a run with a zero balance MUST leave the allocation untouched, and a run with a
> negative balance - the payment now covers less than it is allocated to - MUST release the excess
> so that the allocations sum to the payment again. The excess MUST be released from the most
> recently created allocations first, reducing rather than removing the last allocation it touches
> when only part of it is excess. Releases are ordinary junction writes, so the invoice `paid`
> roll-up follows them. Because every run recomputes, a re-delivered or replayed change MUST leave
> the allocation unchanged.

Allocation remains eventually consistent, not transactionally exact, under concurrency.
