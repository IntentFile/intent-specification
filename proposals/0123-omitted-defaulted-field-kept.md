# An update that leaves out a defaulted field keeps the stored value

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/163, https://github.com/eclipse-dirigible/dirigible/issues/8105
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8250

## The problem

An update replaces the whole record, and `defaultValue` / `init` apply on **create** only. So a
client that sends only what it changed, or a form that does not render the field, writes the field
empty:

- a ticket's priority, seeded by `init`, goes empty and the priority report shows a blank bucket;
- an employee's `active` (default `true`) goes empty, and a `forbidWhen "Employee.active == false"`
  then lets a departed employee's claim through;
- a `required` boolean with a default answers the update with a server error instead of a
  validation outcome.

1.11 says a generator "MUST apply the default when creating a new record and MUST NOT re-apply it
to an existing one". It does not say what an update that leaves the field out does to the stored
value, and the natural reading of a whole-record update - the field is now empty - is the one that
breaks every rule reading it.

## The proposed shape

No new key. The fields are the ones 1.11 already has:

```yaml
fields:
  - { name: active,   type: boolean, defaultValue: true }
  - { name: internal, type: boolean, required: true, defaultValue: false }
relations:
  - { name: Priority, kind: manyToOne, to: TicketPriority, init: 2 }
```

## Expected behaviour

- An update that leaves out a field carrying `defaultValue`, or a relation carrying `init`, keeps
  the stored value; such a field never becomes empty on update, and so cannot be cleared by one.
- A value the update does carry still replaces the stored one.
- The kept value is what the update's checks and guards read.
- An optional field without a default is still cleared when left out, and a required field without
  a default that is left out is still refused, naming it.

## Edge rules

- An update that sends such a field without a value is treated as leaving it out: the stored value
  is kept.
- The default is still not re-applied: what is kept is the stored value, whatever it is.
- A field the application owns (read-only, aggregate) was already kept from the stored row and is
  unaffected.
- A model without `defaultValue` or `init` behaves as before.

## Prior art / workarounds

Clients re-sending every field they did not change, or a form rendering a hidden input for each
defaulted field. Reference implementation: https://github.com/eclipse-dirigible/dirigible/pull/8250.

## Specification text

**Anchor:** Entities & fields > defaultValue - field defaults, a normative block appended after the
first normative block (before "A temporal field may default to the moment the record is made").
**Amends** that first block's "a value the user cleared is a value the user chose": on an update a
defaulted field is replaced, not cleared.

A default is a starting value, and the stored value it started is the one the record keeps: an
update that does not mention the field does not empty it.

> **Normative.**
> An update that leaves out a field carrying `defaultValue`, or a relation carrying `init`, or sends
> it without a value, MUST keep the stored value; such a field never becomes empty on update, and so
> cannot be cleared by one. The kept value MUST be what the update's checks and guards read. A value
> the update carries MUST replace the stored one, as before. An optional field without a default is
> still cleared when left out, and a required field without a default that is left out is still
> refused.
