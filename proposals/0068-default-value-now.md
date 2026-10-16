# `defaultValue: now` on a date, timestamp, month or week field

- **Status:** released in [1.11](../versions/1.11.md)
- **Issue:** https://github.com/IntentFile/intent-specification/issues/104, https://github.com/eclipse-dirigible/dirigible/issues/7603, https://github.com/eclipse-dirigible/dirigible/issues/7392
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7627, https://github.com/eclipse-dirigible/dirigible/pull/7395

## The problem

[0038](0038-what-a-copy-does-not-carry-over.md) built a copy's `defaults: { date: now }` on the
observation that "the format has no `defaultValue: now` for a date field". That gap is older than the
copy: an invoice's date, a timesheet's day, a payroll run's month default to the current moment in
every application, and the only tools are a literal (wrong tomorrow) or a create-time calculated
field, which is a hand-written answer to a one-word question. The reference implementation now accepts
`now` as a field default, so 0038's sentence is false and the format is silent on what a conforming
generator does.

## The proposed shape

```yaml
fields:
  - { name: date,      type: date,      defaultValue: now }
  - { name: recordedAt, type: timestamp, defaultValue: now }
  - { name: period,    type: month,     defaultValue: now }
  - { name: week,      type: week,      defaultValue: now }
```

## Expected behaviour

`now` on a temporal field is the current moment in the field's own shape - `YYYY-MM-DD` for `date`,
a full timestamp for `timestamp`, `YYYY-MM` for `month`, `YYYY-Www` for `week` - the rule 0038
states for a copy and 0003 states for rendering. It has `defaultValue`'s three effects: a record
created without the value receives it, it satisfies `required`, and it seeds the new row in the
editor. It is resolved against the acting user's local calendar when a person creates the record, and
against the system clock when a generator does.

## Edge rules

- `now` on a field of any other type is an authoring error.
- The column default is left to the implementation (a database's "current date" function where one
  exists); the specified behaviour is the seeded value, not the DDL.
- Like every default it is a starting value: a cleared value is a chosen value, and the next edit does
  not re-default it.
- 0038's statement that the format has no `defaultValue: now` for a date field is withdrawn; its
  `defaults:` keeps accepting `now` with the same shape rule.

## Prior art / workarounds

A create-time calculated field, or a hand-edited form. Reference implementation: https://github.com/eclipse-dirigible/dirigible/pull/7627,
building on https://github.com/eclipse-dirigible/dirigible/pull/7395 (one shape rule for every temporal kind).

## Specification text

**Anchor:** Entities & fields > defaultValue - field defaults, after the normative block.

A temporal field may default to the moment the record is made:

```yaml
- { name: date,   type: date,      defaultValue: now }
- { name: period, type: month,     defaultValue: now }
```

`now` is the current moment in the field's own shape - `YYYY-MM-DD`, a timestamp, `YYYY-MM`,
`YYYY-Www` - resolved in the acting user's calendar.

> **Normative.**
> A conforming generator MUST accept `defaultValue: now` on a `date`, `timestamp`, `month` or
> `week` field and MUST reject it on any other type. It MUST seed a new record with the current
> moment rendered in the field's own shape, resolved against the acting user's local calendar for an
> interactive create and against the system clock otherwise, and MUST give it the three effects of
> every `defaultValue`: applied on create, never re-applied, satisfying `required`.

## DSL index

| Construct | What it does |
| --- | --- |
| [`defaultValue: now`](#defaultvalue--field-defaults) | a date / timestamp / month / week field defaults to the current moment in its own shape |
