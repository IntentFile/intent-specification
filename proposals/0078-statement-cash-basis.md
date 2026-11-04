# A statement report on the cash basis - `cashAccounts:` declares the direct-method cash-flow statement

- **Status:** released in [1.12](../versions/1.12.md)
- **Issue:** https://github.com/IntentFile/intent-specification/issues/139, https://github.com/eclipse-dirigible/dirigible/issues/7857
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7880

## The problem

`kind: statement` declares a fixed structure of lines over the balance ledger and expresses the
balance sheet, the income statement and the changes in equity. It cannot express the **cash-flow
statement**. The direct method classifies cash movements by their **counter side** - receipts from
customers are the credits to receivables in entries that debit cash - while every statement line in
1.11 reads an account's own balance: "A line is either a **leaf**, which reads the ledger (`accounts`
selects, `measure` says which balance to take)". No line can ask how much cash an account's
movements brought in or paid out.

## The proposed shape

```yaml
reports:
  - name: CashFlow
    kind: statement
    source: JournalEntryItem
    date: journalEntry.entryDate         # hops to the entry - the attribution is per entry
    debit: debit
    credit: credit
    account: account.code
    cashAccounts: "50*"                  # the accounts selector grammar: which accounts are cash
    lines:
      - { code: A.1, label: Receipts from customers, accounts: "41*", measure: periodCredit }
      - { code: A.2, label: Payments to suppliers,   accounts: "40*", measure: periodDebit }
      - { code: A,   label: Operating activities,    sum: [A.1], less: [A.2] }
```

## Expected behaviour

`cashAccounts:` switches the statement to the **cash basis**, keeping the `lines` / `sum` / `less`
vocabulary:

- Only entries that touch a cash account count.
- Each line of such an entry is attributed its proportional share of the entry's cash movement over
  the **whole** counter side: cash received is the entry's cash debit, spread over its credit side by
  amount; cash paid is its cash credit, spread over its debit side by amount.
- A leaf line's `accounts` are **counter-accounts**: `periodCredit` is the cash received from them,
  `periodDebit` the cash paid to them.
- Over a balanced entry the non-cash shares add up exactly to the entry's net cash movement, so a
  complete statement's sections equal the change in the cash accounts' balance over the window.
- Cash-to-cash transfers are the cash lines' own shares, and drop out: the cash accounts are excluded
  from what the leaf lines read.
- The per-entry cash totals are a fact about the entry: `filter` and lifecycle `scope` restrict which
  rows count, not how an entry's cash is attributed.

## Edge rules

- Each is an authoring error: a malformed `cashAccounts` selector; a `date` that does not hop over a
  to-one relation of the source (the entry); a measure outside the four `period` forms, since an
  opening or closing cash flow does not exist; `cashAccounts` on anything but a `kind: statement`.
- A statement without `cashAccounts` reads and generates as in 1.11.

## Prior art / workarounds

None within the format: every statement line read an account's own balance, so the cash-flow
statement was a hand-written query over the journal. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/7880.

## Specification text

**Anchor:** Presentation > reports > statement reports - account-to-line mappings, a new subsection
after its normative block.

#### The cash basis - `cashAccounts`

The direct-method cash-flow statement classifies cash by where it came from and where it went - the
**counter side** of the entries that move cash. `cashAccounts` names the cash accounts, in the
`accounts` selector grammar, and switches a statement to the **cash basis**; the line vocabulary is
unchanged:

```yaml
reports:
  - name: CashFlow
    kind: statement
    source: JournalEntryItem
    date: journalEntry.entryDate         # its first hop is the entry the lines share
    debit: debit
    credit: credit
    account: account.code
    cashAccounts: "50*"
    filter: "journalEntry.status == 2"
    lines:
      - { code: A.1, label: Receipts from customers, accounts: "41*", measure: periodCredit }
      - { code: A.2, label: Payments to suppliers,   accounts: "40*", measure: periodDebit }
      - { code: A,   label: Operating activities,    sum: [A.1], less: [A.2] }
```

On the cash basis only the entries that touch a cash account count, and a leaf line's `accounts`
are **counter-accounts**: `periodCredit` is the cash received from them, `periodDebit` the cash paid
to them. A line of such an entry is attributed its proportional share of the entry's cash movement,
taken over the whole counter side - cash received is the entry's cash debit, spread over its credit
lines by amount, and cash paid is its cash credit, spread over its debit lines by amount. Over a
balanced entry the shares of the non-cash lines add up to exactly the entry's net cash movement, so
the sections of a complete statement equal the change in the cash accounts' balance over the window;
that reconciliation is the property to check when in doubt. A transfer from one cash account to
another is the cash lines' own share, and drops out, because the cash accounts are not read by the
leaf lines. The entry is the entity the first hop of `date` reaches, as for a
[correspondence](#balance-reports).

> **Normative.**
> `cashAccounts` is a selector in the `accounts` grammar, and is admitted only on a
> `kind: statement`. With it declared, only the entries that hold a line on a cash account count; a
> line of such an entry MUST be attributed the entry's cash debit times its credit amount over the
> entry's credit total, and the entry's cash credit times its debit amount over the entry's debit
> total; and the cash accounts' own lines MUST NOT be read by a leaf line. `periodCredit` MUST be the
> cash received from the selected accounts and `periodDebit` the cash paid to them. For a balanced
> entry the attributed amounts of its non-cash lines MUST sum to its cash debit less its cash credit,
> so that a statement whose lines cover every non-cash account equals the change in the cash
> accounts' balance over the window. A generator whose cash-flow statement does not reconcile with
> the change in cash is non-conforming.
> An entry's cash totals are a fact about the entry: `filter` and `scope` select the reported rows
> but MUST NOT change how an entry's cash is attributed.
> Each of the following MUST be rejected as an authoring error: a malformed `cashAccounts`
> selector, by the rules that apply to `accounts`; a `date` that is not a `<relation>.<field>` path
> over a to-one relation of the source; a `measure` that is not one of the four `period` forms; and
> `cashAccounts` on a report that is not a `kind: statement`. A statement without `cashAccounts` is
> read on the accrual basis, as before.

## DSL index

| Construct | What it does |
| --- | --- |
| [`kind: statement` / `cashAccounts`](#the-cash-basis--cashaccounts) | the cash basis - the direct-method cash-flow statement: lines read the cash received from and paid to their counter-accounts, reconciling with the change in cash |
