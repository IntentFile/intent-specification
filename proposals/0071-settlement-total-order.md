# A settlement's order is total - `order` takes a list, and the key breaks the ties

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/107, https://github.com/eclipse-dirigible/dirigible/issues/7556
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7661

## The problem

A settlement allocates a payment over the payer's open invoices "in `order`", and `order` names one
field: `order: date`. Every invoice issued on the same day ties, and the format says nothing about
ties, so they come back in storage order - which differs between databases and changes when a row is
updated. A payment that covers one of two same-day invoices pays the newer one in full and leaves the
older one PARTIAL, and the same file settles differently on two deployments.

## The proposed shape

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
    order: [date, number]       # a scalar still works: order: date
    match: [Customer, Currency]
    status: Status
    payableStatuses: [3, 4, 6]
```

## Expected behaviour

`order` takes a list of fields of the settled document; a scalar is the one-element list. The order is
**total**: after the authored fields, the document's primary key ascending - creation order - breaks
every remaining tie, and it is not repeated when the author already named it. Both directions sort the
same way: the open invoices a payment is spread over, and the payments an invoice draws from when it
is the side that changed.

## Edge rules

- Every name must be a field of the invoice entity; an empty list and an unknown name are authoring
  errors.
- A to-one relation is not an order field: ordering by a related record's attribute is not specified.
- Releasing excess (0021) keeps its own rule - most recently created allocation first - which is the
  reverse of the same key, so the two orders agree.

## Prior art / workarounds

None at the model level. Reference implementation: https://github.com/eclipse-dirigible/dirigible/pull/7661.

## Specification text

**Anchor:** Declarative glue > settlements - payment allocation. The example's `order: date` line
gains the comment `# or a list: [date, number]`; the following is appended after the normative block.

`order` MAY be a list of the settled document's fields; a scalar is the one-element list. The order
it defines is total: the document's primary key, ascending, breaks the ties the authored fields leave.

> **Normative.** A conforming generator MUST order the open documents by the authored `order`
> fields in sequence, ascending, and then by the document's primary key ascending, omitting the key
> where the author already named it, so that two runs over the same rows on any store allocate
> identically. It MUST apply the same order in both directions of the allocation. An empty `order`
> and a name that is not a field of the settled document MUST be reported as authoring errors.

## DSL index

| Construct | What it does |
| --- | --- |
| [`settlements[].order`](#settlements--payment-allocation) | the fields open documents are settled in - a scalar or a list - made total by the primary key |
