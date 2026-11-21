# A posting adopts only a post it created - its back-reference is system-owned

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7825
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8819

## The problem

A [posting](#postings--source-document-to-ledger) finds "its" post through the `backReference` - the
created document's to-one back to the source - and, since an amended source rewrites its post, it
rewrites a post that is still in its create status in place: the header assignments re-applied, the
items replaced by the derived set.

The back-reference is an ordinary relation of the created document, so a user can set it. An
accountant who writes a manual journal entry and, helpfully, links it to the invoice it concerns has
handed that entry to the posting: the next time the invoice reaches its moment, the posting finds the
entry, sees content that differs from what the invoice implies, and rewrites it. The user's booking
is gone, with no error and no second entry.

[`posts`](#posts--derived-rows-on-an-event) has the same hole the other way round: a user who sets the
`idempotentBy` back-reference on a row makes the rule believe the source is already posted, and it
skips.

## The proposed shape

No new key.

```yaml
postings:
  - name: salesInvoicePosting
    event: { onTransition: SalesInvoice, model: sales-invoices, when: "Status == 3" }
    creates: JournalEntry
    backReference: SalesInvoice          # system-owned: users cannot write it
posts:
  - name: goodsReceiptLedger
    event: POSTED
    into: StockMovement
    idempotentBy: GoodsReceipt           # system-owned: users cannot write it
```

## Expected behaviour

- A posting's `backReference` and a `posts` entry's `idempotentBy` relation are read-only to users,
  as if declared `readOnly`: a user create drops a value sent for it, a user update keeps the stored
  value, and generated forms show it read-only.
- The posting and the `posts` rule themselves still write it, so only a row they created carries it,
  and a posting rewrites only its own post.

## Edge rules

- The rule applies to a back-reference on an entity of the generating model. A target owned by
  another model is that model's to generate.
- Where read-only would contradict what the author declared on the relation - a `required` relation
  no user create could then fill, a composition parent, a `function: EntityStatus` relation, an owner
  or partner relation, or a calculated relation - an implementation is not required to apply the rule.
- A create-from's `backReference` is not concerned: it is a dimension a hand-entered row may carry
  too, and an `undo` over it is meant to cover such rows.

## Prior art / workarounds

Declaring the relation `readOnly` by hand, which most models forget.

## Specification text

**Anchor:** Declarative glue > `postings — source document to ledger`, after the subsection
`An amended source rewrites its posting` and its normative blocks (before `reversals`).

The back-reference a posting finds its post by is the posting's to write. A user cannot point a
document of their own at a source, so a posting never adopts an entry it did not create; the same
holds for a `posts` entry's `idempotentBy`.

> **Normative.**
> A conforming generator MUST treat a posting's `backReference` and a `posts` entry's `idempotentBy`
> relation, on an entity of the generating model, as read-only to users whether or not the author
> declares it: a user create MUST drop a value sent for it, a user update MUST keep the stored value,
> and a generated form MUST render it read-only. The posting and the `posts` rule MUST still write
> it. An implementation is not required to apply this to a relation declared `required`, to a
> composition parent, a `function: EntityStatus` relation, an owner or partner relation, or a
> calculated relation, where read-only would contradict the declaration.

## DSL index

No new row; the `postings` and `posts` rows are unchanged.
