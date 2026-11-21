# A period register partitions time - overlapping windows are refused

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7829
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8821

## The problem

A `period:` register accepts any rows. Two of them can cover the same date - March as one OPEN row,
and a CLOSED row for 15 March to 15 April - and "the period covering this date", which both the
period lock and the reopen sweep of a deferred posting ask, then has no single answer. The format
currently calls overlap "a data question, not a modelling one" and resolves it by freezing when any
covering row is closed. That keeps the lock safe, but a reopened period can leave the same dates
frozen by its overlapping twin, and nobody is told why.

A register is a partition of time by construction. The format can say so.

## The proposed shape

No new key.

```yaml
- name: AccountingPeriod
  period: { start: startDate, end: endDate, closedWhen: "Status == CLOSED" }
```

## Expected behaviour

- A create or update of a register row whose `[start, end]` window overlaps another row's window is
  refused as a conflict naming the overlapped row and its window.
- Both bounds are inclusive, as for the lock: windows sharing a day overlap; windows that only touch
  (one ends on the 31st, the next starts on the 1st) do not.
- The whole register is the scope: a register declares no partition key, as the lock treats a date
  as closed when any covering row is closed.
- Two concurrent writes to a register never both commit overlapping windows.

## Edge rules

- An update is asked only when it moves a bound, so a pair stored before the rule existed stays
  editable in its other fields.
- Rows already overlapping in stored data keep the existing lock semantics: any closed covering row
  freezes the date.
- An implementation is not required to serialise the first two rows written concurrently into an
  empty register.

## Prior art / workarounds

A hand-written check in the escape-hatch folder; or discipline.

## Specification text

**Anchor:** Entities & fields > `immutableInPeriod / period — period locking`, the first normative
block. Two changes.

1. Replace "When more than one row covers a date and any of them is closed, the record is frozen;
   overlapping periods are a data question, not a modelling one." with "When more than one row
   covers a date - rows stored before the register refused overlap - and any of them is closed, the
   record is frozen."

2. Append after that block:

The register partitions time: its windows do not overlap.

> **Normative.**
> A conforming generator MUST refuse a create or an update of a register row whose window overlaps
> another row's window, as a conflict naming the overlapped row and its window, on every writer of
> the register. Both bounds MUST be read inclusively: two windows sharing a day overlap, two windows
> where one ends the day before the other starts do not. The whole register MUST be the scope of the
> rule. An update MUST be asked only when it changes a bound. Two concurrent writes to a register
> holding rows MUST NOT both commit overlapping windows; an implementation is not required to
> serialise the first rows written concurrently into an empty register.

## DSL index

No new row; the `immutableInPeriod` / `period` row is unchanged.
