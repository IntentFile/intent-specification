# An expansion reconciles its child set instead of rebuilding it

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/6817
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/6841

## The problem

1.5 says of `expansions` only that "a span change replaces the generated child set". Read literally,
that permits the simplest implementation: delete every row pointing at the master, then create the
set the new span calls for.

Nothing about a generated application makes that pair of steps atomic. The reaction that maintains
the child set runs as an event handler, and every write it performs commits on its own — there is no
transaction spanning them, which is the same consistency model the format already assumes elsewhere
(a posting is specified as idempotent and resumable, explicitly *not* transactional).

So a failure partway through the recreation is not a retryable no-op. A leave request covering ten
days is edited to twelve: the ten day rows are deleted, the fifth new one fails a validation on the
child, and what remains is four rows where there should be twelve — with a count field stating
something else again, and every roll-up downstream having consumed four deletes and four creates.
Nothing is logged as inconsistent, because from each individual write's point of view nothing went
wrong. The record is quietly wrong in a direction that surfaces at manual reconciliation.

The destructive step is also unnecessary. Most span edits move an edge — a stay extended by a night,
a loan restructured by two months — so the overwhelming majority of the rows deleted are recreated
identically, with new identifiers.

## The proposed shape

No change to the authored YAML — this is a behaviour rule for an existing construct:

```yaml
expansions:
  - name: nights
    from: Stay
    into: StayNight
    between: { start: fromDate, end: toDate }
    map: { day: period }
    spread: { total: total, into: amount, round: 2 }
    count: nights
```

## Expected behaviour

A span change is applied as a **diff** against the rows that exist:

- a period the new span covers that has no row **gains** one;
- a row whose period the new span no longer covers is **deleted**;
- a row whose period the new span still covers is **kept** — the same row, with the same identifier,
  and with whatever was edited on it.

With `spread`, a kept row's share is **recomputed** for the new row count and written to that row:
the share is derived from the master's total and the number of rows, so leaving it at the value the
previous count produced would break the invariant that the shares sum to the total.

The observable consequences: an interrupted reconciliation can leave the set incomplete, but can no
longer destroy a row the span still calls for; and a reference to a generated row — an allocation
against a night, a comment on an installment — survives an edit to the span's other end.

## Edge rules

- **Duplicates are the repair path.** Two rows on the same period (which only an interrupted run or a
  hand-entered row can produce) leave one row for that period; the rest are deleted. This is what
  makes a partially reconciled set converge on the next delivery.
- **A row with no period** is deleted — it is in the set but not of it.
- **Unchanged means untouched.** A span that resolves to the periods already present writes nothing
  at all, which is what bounds the cascade when a roll-up's write-back re-triggers the reaction.
- **Deletes still go through the child's own layer**, so a deleted row's event fires and the
  roll-ups and guards downstream of it run exactly as for a hand-deleted row. Unchanged from 1.5.
- **The expansion still owns the set**, so hand-entered rows in an expanded child remain a mistake: a
  row on an uncovered period is deleted as stale, and a second row on a covered period as a
  duplicate. The diff narrows *when* rows are destroyed, not *whose* rows they are.

## Prior art / workarounds

None available to an author: the reconciliation is generated, so the destructive window cannot be
avoided from the intent. The only workaround was operational — reconcile by hand after an
interrupted edit, if anybody noticed.

A transaction boundary around the whole reconciliation would be a different (and larger) answer, and
one the format's consistency model does not currently offer to generated reactions. The diff needs no
such boundary: it makes the failure mode incomplete rather than destructive.

## Specification text

**Anchor:** Declarative glue > `expansions` — child rows from a date span (replacing the paragraph
"A span change replaces the generated child set — never mix hand-entered rows into an expanded
child.")

A span change is **reconciled** against the rows that exist rather than rebuilt: the periods that are
missing are added, the rows whose period the span no longer covers are deleted, and the rows it still
covers are kept — the same rows, with the same identifiers, and with whatever was edited on them.
Never mix hand-entered rows into an expanded child: the expansion owns the set, so a row on a period
the span does not cover is deleted as stale, and a second row on a covered period as a duplicate.

> **Normative.**
> A generator MUST apply a span change to the generated child set as a diff: it MUST create a row for
> each period the new span covers that has none, MUST delete each row whose period the new span does
> not cover, and MUST NOT delete or recreate a row whose period the new span still covers. Where
> `spread` is declared, it MUST recompute a kept row's share for the new row count. A reconciliation
> that resolves to the set already present MUST write nothing.
>
> The rule exists because the reconciliation's individual writes are not one atomic step: an
> implementation that deletes the whole set before recreating it destroys committed rows whenever the
> recreation is interrupted, whereas a diff leaves the set incomplete at worst — repaired by the next
> reconciliation, which resolves duplicate rows on one period down to one.
