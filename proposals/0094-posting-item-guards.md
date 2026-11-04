# A posting item guard takes the term grammar, an unreadable one is an authoring error, and a rule column is required only where the line that reads it books

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/155, https://github.com/eclipse-dirigible/dirigible/issues/7993, https://github.com/eclipse-dirigible/dirigible/issues/7649
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7996, https://github.com/eclipse-dirigible/dirigible/pull/7680
- **Amends:** [0053 - a row outside every tuple is not guarded, and a rule match is a literal that says something](0053-tuple-less-rows-and-empty-rule-matches.md)

## The problem

**The guard.** 1.11 shows a posting item's `when` only by example - `when: "Vat != 0"` - and states
no grammar. The computed lines of a create-from (`generates` with a list `items`) carry the same row
guard, documented as "optional guard: `<SourceField> ==|!= <number>`". The rule an accountant
actually writes - "debit deductible VAT only where there is a right to tax credit" - has no form
under that reading: a string comparison (`VatTransactionCode != 'NOCR'`), an ordering (`Vat > 0`)
and a second term are all outside it. Worse, the specification does not say what an unreadable
guard means, and the reading "no guard" is available: a guard a generator could not parse was
treated as absent, and the line booked on every document.

**The rule columns.** A posting derives its accounts from a rule row (`rule(<column>)`). Read
literally, every rule column any item reads must be present before anything is derived. So a column
read by one guarded line - a promotion account on the few invoices that carry a promotion - gates
the **whole** posting: the day that line is added, every document of the type stops posting on every
rule row that predates the column, including the documents the line does not apply to. 1.11's "a
missing rule or account skips (an unposted worklist), never throws" is then the whole symptom - the
documents accumulate on the unposted worklist and nothing says why.

## The proposed shape

```yaml
postings:
  - name: salesInvoicePosting
    items:
      - { Account: rule(receivableAccount), debit: "Net + Vat" }
      - { Account: rule(vatAccount),   credit: "Vat",   when: "Vat > 0 && VatTransactionCode != 'NOCR'" }
      - { Account: rule(promoAccount), debit: "Promo",  when: ["Promo != 0", "Region == 'EU'"] }   # a list is the same AND
```

## Expected behaviour

- A row guard is one term, or several joined by `&&`, or a list of terms, which is the same AND. A
  term is `<field> ==|!= '<string>'` (single or double quotes) or
  `<field> ==|!=|<|<=|>|>= <number>`. A number term reads a null or missing field as 0, as a posting
  amount does; a string term is an equality of texts.
- A guard that does not read is an **authoring error**, reported when the intent is read and naming
  the posting (or the create-from) and the item's position. It is never "always".
- Over a source the model can see, a term on a field the source does not declare and a string term
  on a field that is not a string are authoring errors too - either line would silently never book.
  A source owned by another model is shape-checked only.
- A `rule(<column>)` read by an **unguarded** row is required up front: that line books on every
  document, so a null column stops the posting. A `rule(<column>)` read only by **guarded** rows is
  required only where one of those rows' guards holds; a rule row without the column posts the
  documents the line does not apply to.
- Every skip - no matching rule row, a null column, an undetermined account - is reported, naming the
  rule entity, the match value and the source document.

## Edge rules

- A single numeric equality keeps its existing reading, so an existing file generates exactly as
  before.
- The item rows are derived before anything is written, so a column a booking line needs still stops
  the posting before its first write.
- The same grammar guards a computed line of a create-from (`generates` with a list `items`).
- `posts` and the `when` of `transitions` keep their own grammar; neither is concerned.

## Prior art / workarounds

None within the file. A guard outside a single numeric equality was refused, or - by a generator
that could not read it - booked always; and a rule row that predated a column read by one guarded
line stopped every document of its type. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/7996 (guard grammar),
https://github.com/eclipse-dirigible/dirigible/pull/7680 (rule columns).

## Specification text

Three pieces of text, each at its own anchor.

**Anchor:** Declarative glue > postings - source document to ledger, appended after the **Normative**
block on `rule.match` and before the paragraph "The trigger is `onTransition` ...".

An item row's optional `when` is the **row guard**: the line is booked only for a source on which it
holds. It is one term, several terms joined by `&&`, or a list of terms - a list is the same AND:

```yaml
    items:
      - { Account: rule(vatAccount),   credit: "Vat",  when: "Vat > 0 && VatTransactionCode != 'NOCR'" }
      - { Account: rule(promoAccount), debit: "Promo", when: ["Promo != 0", "Region == 'EU'"] }
```

A term compares a field of the source with a literal: `<field> ==|!= '<string>'`, the string in
single or double quotes, or `<field> ==|!=|<|<=|>|>= <number>`. A number term reads a null or
missing field as 0, as an item amount does; a string term is an equality of texts.

> **Normative.** A conforming generator MUST book an item row only for a source on which every term
> of its `when` holds. A guard that is not one or more terms of the grammar above MUST be reported as
> an authoring error when the intent is read, naming the posting and the position of the item; a
> conforming generator MUST NOT treat a guard it cannot read as absent. Where the source is an entity
> of the same model, a term on a field the source does not declare, and a string term on a field
> that is not a string, MUST be reported as authoring errors as well; a source owned by another model
> is checked for the grammar only.

**Anchor:** the same section, appended after the paragraph that ends "... the source document is
skipped to the unposted worklist." (the paragraph 0053 added), as a second paragraph of it.

A rule column is required only where the line that reads it books. A `rule(<column>)` read by a row
with no `when` is needed for every document of the type, so a rule row whose column is empty skips
the source document as a missing rule row does. A column read only by guarded rows is needed only for
the documents on which one of those guards holds: a rule row that predates an optional line keeps
posting every document the line does not apply to. Every skip says why.

> **Normative.** A conforming generator MUST require a rule column read by an unguarded item row
> before anything is written, and MUST require a rule column read only by guarded rows only for a
> source on which one of those rows' guards holds. The item rows MUST be derived before the created
> document is written, so a required column that is empty skips the source before its first write.
> Each skip - no matching rule row, an empty required column, an account that cannot be determined -
> MUST be reported, naming the rule entity, the match value and the source document.

**Anchor:** Declarative glue > generates - create-from, the computed `items` example: its comment
`# optional guard: <SourceField> ==|!= <number>` is amended to
`# optional guard: the posting row-guard grammar`, and the sentence after the example gains, after
"a `when` cell guards the whole line":

A computed line's `when` is the [posting row guard](#postings--source-document-to-ledger), with its
grammar and its authoring errors, the create-from named in place of the posting.

## DSL index

No new row. The `postings` row gains "item `when` row guards: string, ordering and ANDed terms":

| Construct | What it does |
| --- | --- |
| [`postings`](#postings--source-document-to-ledger) | declarative source-document to balanced-document posting (an amended source rewrites the post it already made); item `when` row guards: string, ordering and ANDed terms |
