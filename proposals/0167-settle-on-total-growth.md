# A settlement also runs when the settled document's total grows

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8489
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8839

## The problem

`settlements:` allocates a payer's money to open documents when a payment is written, and when a
document becomes payable. A document's `total:` can grow after that point. A debit note raises a
bill's payable amount, or a charge is rolled up onto an issued invoice. Nothing allocates the money
the payer already has on account to the amount that is now open:

1. A bill of 100.00 is approved. A supplier payment of 150.00 settles it, so the bill is paid and
   50.00 stays on account.
2. An approved debit note raises the bill's total to 110.00, and the bill is partially paid again.
3. The 50.00 on account is not applied to the 10.00 now open. The bill stays partially paid until a
   person allocates by hand or a new payment arrives.

## The proposed shape

No new key. The existing declaration covers it:

```yaml
settlements:
  - name: autoAllocate
    junction: BillSupplierPayment
    invoice: Bill
    payment: SupplierPayment
    amount: amount
    total: payable
    paid: paid
    pot: amount
    order: date
    match: [Supplier, Currency]
    status: Status
    payableStatuses: [3, 4]
```

## Expected behaviour

- When the settled document's `total` grows while the document is in a payable status, the payer's
  unallocated money is allocated to it. The rules are the ones used when the document became
  payable: payments in `order`, matched on `match`, each contributing its unallocated balance, up
  to the document's open amount.
- A change to the document's `paid` amount does not trigger the allocation. That includes an
  allocation a person removed by hand, so such an allocation is not put straight back.
- Repeating the allocation for the same growth changes nothing.

## Edge rules

- A document outside `payableStatuses` is not settled, as before.
- The open amount is the `total` less the amounts already allocated to the document through the
  `junction`. The allocation never exceeds it, and never exceeds a payment's unallocated balance.
- Two allocations to the same document that run at once never spend the same open amount twice.

## Prior art / workarounds

A hand-written listener on the document's change of the column that feeds its total, which
allocates the payer's money on account to the reopened document. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8839.

## Specification text

**Anchor:** Declarative glue > settlements, appended after the first normative block (the one on
the payment's creation and changes).

A document's `total` can grow after the document became payable, for example through a debit
note or a charge added to an issued document. The money its payer already has on account then
settles the amount that opened.

> **Normative.** A conforming generator MUST run the document-side allocation when the settled
> document's `total` increases while the document is in one of `payableStatuses`, by the same rules
> as when the document became payable: payments in `order`, matched on `match`, each contributing
> its unallocated balance, up to the document's open amount, the `total` less the amounts already
> allocated to it through the `junction`. A change of the document's `paid` amount alone MUST NOT
> trigger it, so an allocation removed by hand is not restored. A repeated run MUST leave the
> allocation unchanged, and two runs for the same document MUST NOT together allocate more than its
> open amount.

## DSL index

No new row: the `settlements` row stands.
