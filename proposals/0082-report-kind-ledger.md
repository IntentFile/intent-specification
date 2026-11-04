# A ledger report lists one account's lines with the opening balance, a running balance and the closing balance - `kind: ledger`

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/143, https://github.com/eclipse-dirigible/dirigible/issues/7887
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7898

## The problem

`kind: balance` aggregates per dimension and `kind: statement` renders fixed lines. Neither lists the
**lines** of one account, and a plain report carries no running total. The account analysis - the
account card an accountant reconciles with: the opening balance, each line with its running
balance, the closing balance - cannot be declared in 1.11, which has no `kind: ledger`.

## The proposed shape

```yaml
reports:
  - name: AccountAnalysis
    kind: ledger
    source: JournalEntryItem
    date: JournalEntry.date
    debit: debit
    credit: credit
    account: Account                  # the to-one relation; bound at run time as the `account` parameter
    columns: [JournalEntry.number, JournalEntry.Reason, Customer, Supplier, note]
    correspondence: Account.number    # optional: each line's opposite-side accounts of the same entry
    filter: "JournalEntry.Status == POSTED"
```

## Expected behaviour

The report declares `fromDate`, `toDate` and `account` (the account's key) on its own behalf. Its
rows are:

- an **opening balance** row - everything before `fromDate`;
- the window's lines in date, entry and line order, each with its own debit and credit, its
  `columns`, and a **running balance**: the opening balance plus the signed sum of the lines so far;
- a **closing balance** row with the window's turnover.

The windows are the balance report's, so opening + debit - credit = closing, and that figure equals
what `kind: balance` shows for the same account and window. The result is pageable, and every page
carries the running balance of the whole window, not of the page. When the account entity declares
`hierarchy:`, a group account lists the lines of its whole subtree. `correspondence:` lists, per
line, the distinct accounts on the opposite side of the same entry. With no account chosen the card
is empty.

## Edge rules

- Each is an authoring error on a ledger: `dimensions`, `measures`, `lines`, `cashAccounts`,
  `widget` and `chart`; an `account` that is not a to-one relation; a source without a primary key;
  an authored parameter named `account`.
- `columns:` on a report of any other kind is an authoring error.
- A hierarchy walk is depth-capped, so a cyclic chart of accounts cannot recurse forever.
- Known gap: report parameters have no entity picker, so `account` is read as a plain input.
- A report without `kind: ledger` generates as in 1.11.

## Prior art / workarounds

A plain report with the account as a parameter, which lists the lines but carries no opening
balance and no running total; or a hand-written query. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/7898.

## Specification text

**Anchor:** Presentation > reports > statement reports - account-to-line mappings, a new section
after it (before *Dashboard KPI widgets*); and an amendment to the normative block of balance
reports.

#### ledger reports - one account's lines

A balance report says how much moved on each account; the account card shows **how**: one
account's lines in date order, between its opening and its closing balance, each line carrying the
running balance it leaves. `kind: ledger` declares it over the same signed ledger a balance report
reads:

```yaml
reports:
  - name: AccountAnalysis
    kind: ledger
    source: JournalEntryItem
    date: JournalEntry.date
    debit: debit
    credit: credit
    account: Account                  # the to-one relation to the account - the card's subject
    columns: [JournalEntry.number, JournalEntry.Reason, Customer, Supplier, note]
    correspondence: Account.number    # optional
    filter: "JournalEntry.Status == POSTED"
```

A ledger declares the ledger inputs of a balance report, plus **`account`** - the to-one relation
to the account whose card it is - and **`columns`**, what each line shows beside its date, debit
and credit. The report declares three parameters on its own behalf: the `fromDate` / `toDate`
window of a balance report, with the same window semantics, and `account`, the key of the account
to show. With no account chosen the card is empty.

The rows are an **opening balance** row, holding everything before the window; the window's lines,
in date, entry and line order, each with its own debit and credit and a **running balance** - the
opening balance plus the signed sum of the lines up to and including it; and a **closing balance**
row carrying the window's turnover. Opening plus debit less credit is the closing balance, which is
the figure a balance report shows for the same account and window. When the account entity
declares [`hierarchy`](#hierarchy--leafonly--tree-entities), a group account's card lists the lines
of every account in its subtree.

`correspondence` takes the paths a [balance report's](#balance-reports) does, resolved against the
opposite-side lines of the same entry; on a ledger it lists, per line, the **distinct** accounts on
the counter side, rather than allocating the line's amount over them.

> **Normative.**
> A ledger's running balance MUST be the opening balance plus the signed sum of the window's lines
> up to and including the line, in date, entry and line order; its opening, turnover and closing
> figures MUST be computed over the windows a balance report uses, so that, for every account and
> every window, the closing balance equals the opening balance plus the window's debit less its
> credit, and equals the closing figure a balance report over the same source, `filter` and `scope`
> shows for that account. A ledger that does not reconcile with its trial balance is
> non-conforming.
> The result MAY be read in pages; every page MUST carry the running balance of the whole window.
> When the account entity declares a hierarchy, a group account MUST list the lines of its whole
> subtree, and the walk MUST be bounded in depth so that a cyclic hierarchy terminates.
> Each of the following MUST be rejected as an authoring error: `dimensions`, `measures`, `lines`,
> `cashAccounts`, `widget` or `chart` on a `kind: ledger`; an `account` that is not a to-one
> relation of the source; a ledger source that declares no primary key; an authored parameter named
> `account` on a ledger; and `columns` on a report that is not a `kind: ledger`.

*balance reports* - the last sentence of the normative block is **amended**: `correspondence` on a
`kind: ledger` report is admitted, with the listing meaning given under
[ledger reports](#ledger-reports--one-accounts-lines), and remains an authoring error on a
`kind: statement` and on a report that declares no ledger `kind`.

## DSL index

| Construct | What it does |
| --- | --- |
| [`kind: ledger`](#ledger-reports--one-accounts-lines) | the account card - one account's lines in date order between its opening and closing balance, each with its running balance, reconciling with the balance report |

The `reports` row gains "ledger reports".
