# Financial statement definitions: declarative account-to-line mappings over the balance ledger

- **Status:** released in [1.6](../versions/1.6.md)
- **Issue:** [eclipse-dirigible/dirigible#6909](https://github.com/eclipse-dirigible/dirigible/issues/6909)

## The problem

A statutory annual statement — a balance sheet, an income statement — is a **fixed line structure**.
Every line is a formula over the chart of accounts: a set of accounts, a balance taken from them, and
a sign rule; some lines are arithmetic over other lines. The set of lines is prescribed by the
jurisdiction, the mapping from accounts to lines is prescribed by the chart, and neither is derivable
from the data.

The format can already describe the numbers this reads from. A **balance report** produces opening /
period / closing debit and credit totals over a signed ledger, per dimension, between two runtime
dates. What it cannot produce is a *statement*: its output is one row per dimension value, so there is
no way to say

- *this line is accounts `20*` and `21*`, netted to the debit side*,
- *this line is the sum of the two above it*,
- *and the lines come in this order, whether or not the ledger has anything in them*.

An account-level balance and a statement line are different shapes, and the gap between them is the
whole mapping. A balance report grouped by account code is the raw material; turning it into a
balance sheet is currently done outside the format — in a hand-written query, a spreadsheet, or a
report the developer writes by hand against the generated tables. That is the last ledger report
family with nothing in the format behind it, and the one most tied to a *declaration* rather than to
code: a jurisdiction change or a chart extension is a mapping edit, not a program change.

## The proposed shape

```yaml
reports:
  - name: BalanceSheet
    kind: statement
    source: JournalEntryItem            # the ledger line items - as for kind: balance
    date: journalEntry.entryDate        # the date the runtime From/To pickers apply to
    debit: debit
    credit: credit
    account: account.code               # the account CODE the lines select on
    filter: "journalEntry.status == 2"  # only posted entries count
    lines:
      - { code: A.I,  label: Fixed assets, accounts: "20*,21*", measure: closingNetDebit }
      - { code: A.II, label: Receivables,  accounts: "41*",     measure: closingNetDebit }
      - { code: A,    label: Total assets, sum: [A.I, A.II] }
      - { code: B.I,  label: Payables,     accounts: "40-49",   measure: closingNetCredit }
      - { code: B,    label: Net assets,   sum: [A], less: [B.I] }
```

A statement declares the same ledger inputs as a balance report, plus two things: **`account`**, the
field holding the account code the lines select on, and **`lines`**, the statement's structure.

A line is either a **leaf**, which reads the ledger (`accounts` selects, `measure` says which balance
to take), or **computed**, which is arithmetic over other lines of the same statement referenced by
their `code` (`sum` adds, `less` subtracts). Never both.

**`accounts`** is a comma-separated selector over the account code:

| term    | selects                                                                                   |
| ------- | ----------------------------------------------------------------------------------------- |
| `20*`   | every account whose code starts with `20`                                                  |
| `4110`  | exactly that account                                                                       |
| `60-69` | every account whose code starts inside the range, the bounds being equally long prefixes   |

**`measure`** is one of twelve: `opening` / `period` / `closing`, each in a `Debit`, `Credit`,
`NetDebit` and `NetCredit` form.

## Expected behaviour

A conforming generator produces a report whose rows are the declared lines, in the authored order,
with three columns — the line's **code**, its **label**, and its **amount** — and the same pair of
runtime From/To date parameters a balance report declares, with the same window semantics (opening
strictly before the start, the period inclusive of both bounds, closing up to and including the end).

The **`Net` measures net an account's two sides before the line sums it**, keeping only what remains
on the named side. This is the sign rule the statement form needs and it is observable, not an
implementation detail: a settlement account carrying more credits than debits contributes to a
`closingNetCredit` line and contributes *nothing* to a `closingNetDebit` line, so the same account can
appear in both the asset and the liability section of one statement and land in whichever one its
actual balance puts it. The netting is **per account**: it happens before the line's sum, never after
it. A line summing raw debits and raw credits reports gross turnover, which is a different — and
occasionally desired — figure, and that is what the four plain measures are for.

Computed lines compose: a subtotal may reference another subtotal, and the amount of a computed line
is exactly the signed sum of the lines it names, evaluated on the same window as its leaves.

The statement's numbers are the format's contribution. **The legally mandated print layout is not** —
it stays a hand-authored print template over the result, which is where the boundary between what the
model expresses and what a document designer expresses already sits.

## Edge rules

Each of the following MUST be an authoring error, reported at generation:

- **`dimensions` or `measures` on a statement.** Its rows are its lines; a dimension would multiply
  every line by the dimension's values and line codes would stop identifying a row.
- **A line that is both a leaf and computed**, or neither. A line that both selects accounts and sums
  other lines counts the same amount twice, silently.
- **A duplicate line `code`**, a `sum`/`less` reference to a code the statement does not declare, and
  a **cycle** in the references.
- **An unknown `measure`.**
- **A malformed selector term**: a range whose bounds are of different length (the comparison is over
  equally long prefixes, so the lengths are what give the range its meaning), a range that ends before
  it starts, an empty term.
- **`account` that does not resolve to a string field**, and `date` / `debit` / `credit` failing the
  rules a balance report already applies to them.
- **`account` or `lines` without `kind: statement`**, and `kind: statement` without `lines`.

Two rules are about ordering rather than validity, and both are observable:

- The lines are rendered in the **authored order**, not sorted by code. Statutory line codes do not
  sort lexicographically into their own structure (`A.II` precedes `A.X` in the form and follows it
  alphabetically).
- A line whose accounts hold nothing renders as **zero**, not as a missing row. The structure is the
  statement; an empty line is information.

The account code an author writes into a selector is drawn from a **closed character set** — letters,
digits, dot and underscore — with the hyphen reserved as the range separator and a trailing asterisk
as the prefix marker. A code outside it is an authoring error rather than a value passed through to
the query.

`filter` and any lifecycle `scope` restrict **which ledger rows count**, exactly as for a balance
report, and apply before the per-account reduction.

## Prior art / workarounds

Mainstream ERPs model statements as a *statement definition* — configuration, not code — for the
reason this proposal exists: the mapping differs per jurisdiction and per chart, and it is maintained
by accountants rather than developers.

Today an application built on the format reaches the same place by dropping out of it: a hand-written
query against the generated ledger tables, or a report assembled in a spreadsheet from an exported
account-level balance. Both re-implement the window and the filter the format already describes and
add only the mapping — which is precisely the part that deserves to be declared.

Whether a statement's lines should additionally be maintainable as **data** — seeded rows a tenant
edits in the running application, rather than authored in the intent — is a genuine second question.
It is deliberately not part of this proposal: the authored form is what makes a statement reviewable,
diffable and versioned with the model, and a tenant-editable variant can be layered over the same
semantics later without changing anything stated here.
