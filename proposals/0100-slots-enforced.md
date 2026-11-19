# A slots window is enforced on every write, and `resource:` names what a slot is booked per

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/161, https://github.com/eclipse-dirigible/dirigible/issues/8097
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8474

## The problem

1.11 shows the slots descriptor as `slots: { start: startTime }` and nothing more. Implementations
already read a booking window from it - opening and closing hours, a slot length, closed weekdays -
but only to draw the slot picker. The generated endpoints knew nothing of the window: a request
stored a Sunday 03:00 appointment, an off-grid 10:07 one, and a second booking of a slot the picker
showed as taken. Applications restated the window as `checks:`, and the two copies drifted.

An application that books one slot per employee or per room has no way to say so; it declared a
`unique: [Employee, at]`, which leaves an appointment with no employee unchecked.

## The proposed shape

```yaml
- name: Appointment
  view: slots
  slots:
    start: at                       # a timestamp field
    open: "09:00"
    close: "18:00"                  # exclusive, as on the picker
    step: 30                        # minutes, counted from open
    disabledDays: [0, 6]            # weekdays, Sunday = 0
    resource: Employee              # optional: a to-one; a slot is booked per value of it
```

## Expected behaviour

- A create or an update whose `start` falls on a `disabledDays` weekday, outside `open`..`close`
  (`close` exclusive), or off the `step` grid counted from `open` (seconds zero) is refused with 400.
  The refusal is a translatable message under the entity's checks, with `{open}`, `{close}` and
  `{step}` filled in.
- A write whose `start` another row already holds is refused with 409. Booked means what the picker
  shows as booked: every row at that start, whatever its status, so a cancelled booking keeps its
  slot until it is deleted or moved.
- With `resource:`, a slot is taken only by a row with the same value of that relation; an empty
  value matches an empty value. Without it a slot is booked as soon as any row holds it.
- An update is checked only for what it changes: the window when the start moved, the booking when
  the start or the resource changed. An edit that leaves a booking where it is never collides with
  itself, and a row stored before the rule existed stays editable.
- The window is read in the time zone the request names, else in the server's. The booked check
  compares instants and does not depend on the zone.
- The rule holds on every surface that writes the entity - the regular, the personal and the
  partner one.
- System writes - a create-from, an import, a schedule, a process step - are not checked, as with
  ungated `checks:`.

## Edge rules

- Only a `timestamp` start is checked; a `date` start has nothing to place on a grid.
- Refused when the intent is read: an `open` or `close` that is not `HH:MM`, a `step` below 1, and a
  `resource` that is not a to-one relation of the entity.
- **A behaviour change** for an application that books one slot per resource: it now declares
  `resource:`, which replaces a `unique: [<resource>, <start>]` and also covers the empty-resource
  case. Without it, a parallel booking of the same start for a second resource is refused with 409.
- The picker is unchanged and still marks every booking, so with a `resource:` it is never looser
  than the server.
- Two concurrent bookings of one free slot can both pass - the same limit a `single:` rule (proposal 0091)
  has.

## Prior art / workarounds

The window restated as `checks:` beside the `slots:` block, and a `unique:` over the resource and the
start for the booking. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8474.

## Specification text

**Anchor:** Presentation > view - calendar, range, slots, appended after "`view: calendar` is also
expressible as the role alias `function: Calendar`."

#### slots - the booking window, and what the server enforces

The slots descriptor names the start a booking is placed on and, optionally, the window it may be
placed in:

```yaml
- name: Appointment
  view: slots
  slots:
    start: at                       # required: the date / timestamp field a booking starts at
    open: "09:00"                   # the first slot of a day
    close: "18:00"                  # the end of the day's slots, exclusive
    step: 30                        # the slot length in minutes, counted from open
    disabledDays: [0, 6]            # weekdays without slots, Sunday = 0
    resource: Employee              # optional: a to-one relation a slot is booked per
```

The window draws the slot picker, and it is also a rule on the records. A create or an update
whose start falls on a disabled weekday, outside `open`..`close`, or off the `step` grid counted
from `open` is refused, as is one whose start another row already holds. Booked means what the
picker shows: every row at that start, whatever its status - a cancelled booking keeps its slot
until it is deleted or moved. `resource:` narrows the booking to rows with the same value of that
relation, so one slot can be booked once per employee or per room; an empty value matches an empty
value. Without `resource:` a slot is booked once, whoever holds it.

An update is checked only for what it changes - the window when the start moved, the booking when
the start or the resource changed - so a booking left where it is never collides with itself, and a
row stored before the window was declared stays editable. The window is read in the time zone the
request names, and in the server's when it names none; the booked check compares instants. As with
ungated [checks](#checks--declarative-validations), a system write - a create-from, an import, a
schedule, a process step - is not held to the window.

> **Normative.**
> `open` and `close` MUST be `HH:MM` times and `step` a whole number of minutes of at least 1;
> `resource` MUST name a to-one relation of the entity; each violation MUST be reported as an
> authoring error. On a `timestamp` start, a conforming generator MUST refuse every user write -
> on every surface that writes the entity - whose start falls on a `disabledDays` weekday, before
> `open`, at or after `close`, or off the `step` grid counted from `open` with zero seconds, with a
> 400 and a translatable message under the entity's checks carrying `{open}`, `{close}` and
> `{step}`; and MUST refuse with a 409 a write whose start another row already holds - every row,
> whatever its status, or with `resource:` every row with the same value of that relation, an empty
> value matching an empty value. An update MUST be checked against the window only when its start
> changed, and against the booking only when its start or its resource changed. The window MUST be
> read in the time zone the request names, falling back to the server's; the booking MUST be
> compared by instant. A `date` start MUST NOT be checked. System writes are exempt.

## DSL index

The `view` row becomes:

| Construct | What it does |
| --- | --- |
| [`view`](#view--calendar-range-slots) | an additional calendar / range page, or a slot-booking page whose `slots:` window (`open` / `close` / `step` / `disabledDays`, booked per `resource`) is enforced on every write |
