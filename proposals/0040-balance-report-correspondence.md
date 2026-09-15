# A balance report buckets turnovers by the corresponding account

- **Status:** draft
- **Issue:** [eclipse-dirigible/dirigible#6908](https://github.com/eclipse-dirigible/dirigible/issues/6908)
- **Implementation:** [eclipse-dirigible/dirigible#6927](https://github.com/eclipse-dirigible/dirigible/pull/6927)

## The problem

A `kind: balance` report answers *how much moved on an account*: opening, period and closing debit
and credit totals per dimension, over a signed ledger, between two runtime dates. Its dimensions are
paths resolved from the source row - `account.code` for a trial balance, `customer` and
`account.code` for a counterparty subledger - and that covers every report whose grouping key is
*on the line*.

The double-entry **general ledger** (главна книга) asks a second question the same shape cannot
express: per account, *against which accounts* did the turnover move. Account 411 Customers' debit
turnover of 510 is one figure on the trial balance; the general ledger shows it as 342 in
correspondence with 702 Revenue and 118 in correspondence with 4532 VAT on sales, plus 50 that
faced no account at all. The corresponding account is not a property of the source row: it sits on
a **sibling line of the same journal entry**, on the opposite side. No `relation.field` path from a
line reaches its sibling, so no dimension can name it, and the standard companion of the trial
balance falls outside the format.

## The proposed shape

One key on a `kind: balance` report:

```yaml
reports:
  - name: GeneralLedger
    kind: balance
    source: JournalEntryItem
    date: journalEntry.entryDate           # its FIRST HOP is the document the lines share
    debit: debit
    credit: credit
    dimensions: [account.code, account.name]
    correspondence: account.code           # bucket the counter-side lines of the same entry by this
    filter: "journalEntry.status == 2"
```

`correspondence` takes the same shapes a dimension does - a field of the source, a one-hop
`relation.field` path, or a bare to-one relation - but is resolved against the **counter-side
sibling line** rather than the line itself, and becomes one more grouping column after the declared
dimensions.

The document whose lines correspond is not declared separately: it is the entity the **first hop of
`date`** reaches. A general ledger takes its date from the journal entry or voucher the lines belong
to, and that relation is exactly the key the siblings share, so a correspondence report's `date`
MUST be a `relation.field` path over a to-one relation of the source to its document - a line-local
date column names no document and is refused.

## Expected behaviour

A conforming generator produces a balance report whose rows are **one per combination of the
declared dimensions and the correspondence bucket**, carrying the same six windowed totals a plain
balance report carries (opening, period and closing debit and credit) and the same runtime From/To
window. The bucket column is labelled from the path prefixed with "correspondent" - `account.code`
renders as *Correspondent Account Code* - and is translated and sorted like any other dimension.

For each reported line, the **counter side** is the set of lines of the same document that sit on
the opposite side: a debit line corresponds with the document's credit lines, a credit line with its
debit lines. A line never corresponds with itself, nor with a same-side sibling - a compound entry's
two debit lines are not each other's correspondents.

The line's amount is **allocated proportionally** over its counter-side buckets: a debit line's
share of a bucket is that bucket's credit divided by the document's total credit, and the mirror for
a credit line. Concretely:

- a **simple entry** (one line on at least one side) attributes the full amount to the single
  counter-account;
- a **compound entry** (M debit lines against N credit lines) splits every line by the counter-side
  amounts - each debit line is spread over the N credit buckets in proportion to their credits, and
  each credit line over the M debit buckets in proportion to their debits;
- a **one-sided entry** - a line whose document has nothing on the counter side (an opening entry, a
  single-line adjustment) - keeps its full amount in one **empty bucket**. It MUST NOT drop out of
  the report.

Over the ledger

```
entry 1   Dt 411  100            Ct 702  100
entry 2   Dt 411  300            Ct 702  200   Ct 4532  100
entry 3   Dt 411   50            (no credit line)
entry 4   Dt 411   60  Dt 412 40 Ct 702   70   Ct 4532   30
```

account 411's period debit is reported as 342 against 702 (100 + 200 + 60 × 70/100), 118 against
4532 (100 + 60 × 30/100) and 50 in the empty bucket - 510 in total, which is what the trial balance
shows for 411 over the same window.

That reconciliation is the property that makes the shape worth having: **for every account and every
window, the totals summed across its correspondence buckets equal the plain balance report's
figures**, on the debit side and on the credit side. A general ledger that disagrees with the trial
balance is worse than none, so the empty bucket and the proportional split are not conveniences -
they are what keep the equality.

## Edge rules

- **`date` must reach the document.** With `correspondence` declared, `date` MUST be a
  `<relation>.<field>` path whose first hop is a to-one relation of the source; the relation is the
  document the lines share. A `date` that is a field of the source, or that starts with something
  other than a to-one relation, is an authoring error naming the report, its `date`, and the
  requirement.
- **The source needs a primary key.** A line is excluded from its own bucket by key, so a source
  declaring no `primaryKey` field is an authoring error.
- **The path resolves like a dimension, against the source.** The sibling is another row of the
  source entity, so `correspondence` is checked as a dimension over the source would be: a
  `relation.field` path whose relation is not a to-one relation of the source, a path whose field the
  target entity does not have, and a bare name that is neither a field nor a to-one relation of the
  source are each rejected. A cross-model relation is resolved at generation, like every cross-model
  reference.
- **A subset relation is refused** as the bucket, exactly as it is refused as a dimension - grouping
  by a stored value set groups by the list, not by its members.
- **An empty `correspondence`** (declared with no path) is rejected with a message naming what the
  key expects.
- **Only on `kind: balance`.** `correspondence` on a `kind: statement` report is rejected - a
  statement's rows are its declared lines, so there is no account axis to bucket. `correspondence`
  on a report with no `kind` is rejected together with `date` / `debit` / `credit`, which already
  require a ledger kind.
- **The window is the document's.** The date is read through the document relation, so every line of
  a document falls in or out of the window together; a document is never half inside it.
- **`filter` and lifecycle `scope` select the reported lines, not their counter side.** The counter
  side of a document is every opposite-side line of that document, so a compound entry's shares are
  computed over the whole document even when the report shows only some of its lines. Restricting
  the report to posted entries through the document's status - the usual `filter` - restricts both
  consistently, because the status is the document's.
- **Unbalanced documents reconcile too.** The share is counter amount over counter total, so a line's
  full amount is always attributed, whether or not the document's debits equal its credits; a
  document that does not balance shows the same imbalance in the general ledger as in the trial
  balance, spread over its buckets.
- **A line carrying amounts on both sides** corresponds as a debit line for its debit and as a credit
  line for its credit; a side holding zero pairs with nothing.
- **The bucket adds a grouping, nothing else.** The declared dimensions, the six totals, the From/To
  parameters, any further authored `parameters`, `filter`, `scope` and the totals footer behave as on
  a plain balance report. An author who wants the trial balance next to the general ledger declares
  two reports over the same source; the second is the first plus `correspondence`.

Nothing in the current balance-report text contradicts the shipped behaviour; the construct is an
addition to it.

## Prior art / workarounds

Accounting systems produce the general ledger as a separate, hand-written query - a self-join of the
ledger lines on their document, restricted to the opposite side, with whatever allocation rule the
vendor chose for compound entries. Where the rule is "one mixed bucket" the report stops
reconciling per correspondent; where it is proportional, the arithmetic is repeated in every
vertical that needs it.

Today an application built on the format reaches the same place by dropping out of it: a custom
query over the generated ledger tables, re-implementing the window, the filter and the lifecycle
scope the balance report already declares and adding only the self-join. The reference
implementation ships the construct as described here - a left self-join on the document key
restricted to the opposite side and excluding the line by key, with the proportional allocation
written so that the product precedes the division and no rounding is introduced, and a test that
executes the generated report over all four entry shapes and asserts the reconciliation with the
plain balance.

## Specification text

**Anchor:** Presentation > reports > balance reports (appended after the `TrialBalance` example, before "statement reports — account-to-line mappings")

A trial balance says how much moved on an account; the general ledger says **against which
accounts**. The corresponding account is not on the line - it sits on a sibling line of the same
journal entry, on the opposite side - so no dimension path can reach it. `correspondence` names the
path that buckets those sibling lines:

```yaml
reports:
  - name: GeneralLedger
    kind: balance
    source: JournalEntryItem
    date: journalEntry.entryDate           # its first hop is the document the lines share
    debit: debit
    credit: credit
    dimensions: [account.code, account.name]
    correspondence: account.code           # bucket the counter-side lines of the same entry by this
    filter: "journalEntry.status == 2"
```

`correspondence` takes the shapes a dimension takes - a field of the source, a one-hop
`relation.field` path, a bare to-one relation - resolved against the counter-side line rather than
the line itself, and adds one grouping column after the declared dimensions, labelled from the path
(*Correspondent Account Code*). The document the lines share is the entity the first hop of `date`
reaches, which is why a correspondence report takes its date over the relation to its journal entry
or voucher.

A line's counter side is the opposite-side lines of its document: a debit line corresponds with the
credit lines, a credit line with the debit lines, never with itself or a same-side sibling. Its
amount is allocated over the counter-side buckets in proportion to their amounts - a debit line's
share of a bucket is the bucket's credit over the document's total credit, and the mirror for a
credit line - so a simple entry attributes the full amount to its one counter-account, a compound
entry splits each line by the counter-side amounts, and a line whose document has nothing on the
counter side keeps its full amount in one empty bucket. Each account's totals across its buckets
therefore equal the plain balance report's figures for the same window; that reconciliation is the
property to check when in doubt.

> **Normative.**
> `correspondence` is a path of the same shapes a dimension admits, resolved against the
> counter-side line of the same document, and adds exactly one grouping column to a `kind: balance`
> report; the six windowed totals, the From/To window, `filter`, `scope` and `parameters` are
> otherwise those of a balance report.
> The document is the entity the first hop of `date` reaches. With `correspondence` declared, `date`
> MUST be a `<relation>.<field>` path over a to-one relation of the source; a line-local `date` MUST
> be rejected. The source MUST declare a primary key, by which a line is excluded from its own
> bucket.
> A line's counter side is the opposite-side lines of its document; a same-side sibling MUST NOT
> form a bucket. A line's amount MUST be allocated over its counter-side buckets in proportion to
> the counter-side amounts, and a line with no counter side MUST keep its full amount in one empty
> bucket rather than be dropped - so that, for every account and every window, the totals summed
> across the correspondence buckets equal the totals a balance report without `correspondence`
> shows over the same source, `filter` and `scope`, on both sides. A generator whose general ledger
> does not reconcile with its trial balance is non-conforming.
> `filter` and `scope` select the reported lines; the counter side of a document is every
> opposite-side line of that document.
> Each of the following MUST be rejected as an authoring error naming the report: an empty
> `correspondence`; a path that does not start with a to-one relation of the source, that names a
> field the related entity does not have, or that is neither a field nor a to-one relation of the
> source; a path onto a subset relation; `correspondence` on a `kind: statement` report; and
> `correspondence` on a report that declares no ledger `kind`.

## DSL index

| Construct | What it does |
| --- | --- |
| [`correspondence`](#balance-reports) | the general ledger axis on a balance report - turnovers bucketed by the account on the opposite side of the same entry, allocated proportionally, reconciling with the plain balance |
