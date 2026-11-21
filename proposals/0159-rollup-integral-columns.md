# A sum roll-up over integer and long columns

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7944
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8811

## The problem

The format lets a sum roll-up name any numeric field for `of`, `field`, `capacity` and `balance`, and
a good share of real totals are counts, not amounts: seats booked against a session's seat count,
hours booked against a project's hour budget, units received against units ordered. The text of
`rollups` speaks only of amounts, and says nothing about what a sum into an integer column, or a
capacity check against one, must do - so an implementation that computes everything as decimal and
writes it back unconverted is not visibly wrong against the text, and the generated application does
not build.

## The proposed shape

No new key.

```yaml
rollups:
  - { name: sessionBooked, entity: Booking, via: Session, field: booked,      # integer
      op: sum, of: places, capacity: seats, balance: seatsLeft,              # integer, integer, integer
      message: "Only {remaining} seats left" }
  - { name: projectHours, entity: Timesheet, via: Project, field: actualHours, # long
      op: sum, of: hours, capacity: budgetHours, balance: remainingHours,     # integer, long, long
      enforce: false }
```

## Expected behaviour

- `of`, `field`, `capacity` and `balance` of a sum roll-up may each be `integer`, `long` or a decimal
  type, in any combination; a roll-up over integral columns behaves as one over decimal columns.
- The capacity check, the measured balance and the refusal placeholders (`{capacity}`, `{sum}`,
  `{remaining}`, `{requested}`) work on integral columns.
- A total written into an integral column is compared with the stored value in that column's type,
  so a recompute that changes nothing writes nothing.

## Edge rules

- A cross-model parent's or child's column is typed from its owning model.
- A roll-up over decimal columns generates as before.

## Prior art / workarounds

Declaring every count as a decimal field, which shows "12.00 seats".

## Specification text

**Anchor:** Declarative glue > `rollups — denormalised parent totals`, after the second paragraph
(the one beginning "A count roll-up keeps a counter").

The summed field and the parent's columns may be counts as well as amounts: a session's seats against
the places of its bookings, a project's hour budget against its timesheets' hours.

> **Normative.**
> A sum roll-up's `of`, `field`, `capacity` and `balance` MAY each be an `integer`, `long` or decimal
> field, in any combination. A conforming generator MUST maintain the total, the balance, the capacity
> check and the refusal placeholders over integral columns as it does over decimal ones, MUST write a
> value into an integral column in that column's type, and MUST compare a recomputed value with the
> stored one in the stored column's type, so a recompute that changes nothing writes nothing. A
> column owned by another model MUST be typed from the owning model.

## DSL index

No new row; the `rollups` row is unchanged.
