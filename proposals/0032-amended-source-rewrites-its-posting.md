# An amended source rewrites its posting

- **Status:** released in [1.7](../versions/1.7.md)
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7071

## The problem

`postings:` derives a document from a source document that reaches a moment - the classic case being
a sales invoice that is issued and the journal entry it must produce. The generated handler is
idempotent through the declared `backReference`, so a redelivered event does not post twice.

A source document, however, does not only reach that moment once. The amend path is ordinary and
documented: the invoice is issued, the approver rejects it, the author adds a line, the document is
issued again - same document, same number, new amounts. The moment fires a second time and the
posting sees a post that already exists, so it does nothing. The entry keeps the amounts of the
first issue while the document it references says something else:

```
SalesInvoice SI00000003  issued 1 200.00  ->  JournalEntry 8: Dt 1 200 / Ct 1 000 / Ct 200
rejected, a line added, issued again at 1 260.00
JournalEntry 8 still says 1 200 / 1 000 / 200
```

No second entry (right) and a ledger 60.00 short (wrong), with nothing in the application saying so.
Idempotence answered "have I already handled this source?" when the question is "does the post still
say what the source says?".

## The proposed shape

No new key. This is the behaviour `postings:` already declares, applied to the second occurrence of
the moment.

```yaml
postings:
  - name: salesInvoicePosting
    event: { onTransition: SalesInvoice, model: sales-invoices, when: "Status == Issued" }
    creates: JournalEntry
    backReference: SalesInvoice
    map: { entryDate: date, reason: "Sales invoice {number}" }
    rule: { entity: PostingRule, match: { documentType: "Sales Invoice" } }
    items:
      - { Account: rule(receivableAccount), debit: "Net + Vat" }
      - { Account: rule(revenueAccount),    credit: "Net" }
      - { Account: rule(vatAccount),        credit: "Vat", when: "Vat != 0" }
```

## Expected behaviour

A conforming generator MUST derive the full content the source implies - the header assignments and
every item row whose guard holds - and compare it with the post the `backReference` finds, before
deciding what to write:

- **no post** - create it, as today;
- **a post carrying exactly the derived content** - do nothing (a redelivery);
- **a post carrying anything else** - rewrite THAT post: re-apply the header assignments and replace
  its items with the derived set. One source has at most one post, before and after.

The rewrite is bounded by the created document's own lifecycle, which the posting itself established:
the created document is rewritable while its `function: EntityStatus` relation still holds the
`init:` value the posting's create wrote (or is still empty, where the relation declares no `init:`).
A created document with no status lifecycle is always rewritable - there is no state for anyone to
have acted on.

Once the created document has left that status - it was posted, approved, closed - the generator MUST
NOT rewrite it. It MUST report the divergence, naming both documents, and leave the correction to a
reversing entry (`reverses:`). Overwriting a document somebody has acted on is worse than the
divergence it repairs.

## Edge rules

- The comparison MUST be order-insensitive: the stored rows have no declared order.
- The comparison covers the union of the cells the item rows assign; a property no row assigns is
  absent on both sides and says nothing.
- Numbers compare by VALUE, so a stored amount handed back at a different scale (`1200` vs `1200.00`)
  is not read as a change.
- A half-post - fewer stored items than the derived set, an item write having failed after the
  document was saved - is the same case: the content differs, so it is rewritten and thereby
  completed. This subsumes the existing resumability rule.
- Reversal (`reverses:`) is unaffected: each handler compares only the documents its storno link
  discriminates as its own.
- The rewrite goes through the created document's ordinary write path, so its validations, checks and
  derived fields apply, and its own update event is published - the post really did change.

## Prior art / workarounds

Today the divergence is repaired by hand: void the entry, or correct it in the ledger, having first
noticed it - and nothing surfaces it, because the posting reported success the first time and did
nothing at all the second. The alternative modelling - forbidding the amend path so a rejected
document can only be cancelled and re-created - loses the document number and the audit trail of the
correction, which is the reason the amend path exists.
