# A `uuid` field is issued by the server on a client create

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8485
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8521

## The problem

A `uuid` field is a record's stable external identity: prints, exports and integrations match on
it. The format says only that it is a 36-character string, not who chooses its value. A client that
sent `"uuid": "not-a-uuid"` on a create, or another record's uuid, had that value stored, so the
identity other systems rely on was the client's to choose, and not even a UUID.

An update already keeps the stored value. The create is the gap.

## The proposed shape

No new key. A field declared

```yaml
fields:
  - { name: uuid, type: uuid, unique: true, major: false }
```

gets a value the server draws on every client create, whatever the payload carries.

## Expected behaviour

- On a create through the generated application's API, on every surface a client creates the record
  through (the regular, personal and partner surfaces), a value the payload carries for a `uuid`
  field is ignored and a fresh UUID is drawn. The create is not refused for it.
- An update keeps the stored value, as before.
- A writer inside the application that creates the record itself - a create-from, a posting, an
  import, an inbound message, hand-written code - gets a drawn UUID unless its own declaration maps a
  value into the field. Such a mapping is the author's, not a client's.
- Data loaded below the application - seed data, a migration - keeps the UUID it carries.
- The field needs no declaration beyond its type.

## Edge rules

- A `uuid` primary key stays refused, as before; this concerns a non-key `uuid` field.
- A `number: { stampOn: issue }` field, whose placeholder is a UUID until the number is drawn, is a
  different field and is not affected.
- A calculated action hosted on the `uuid` field sees no client value on create, exactly as for a
  client that omits the field.

## Prior art / workarounds

A `readOnly: true` on the field keeps it off the forms but did not stop an API client from sending a
value. Reference implementation: https://github.com/eclipse-dirigible/dirigible/pull/8521.

## Specification text

**Anchor:** Entities & fields > fields > Logical types, appended to the paragraph "Generators map
each logical type to a physical column type. ... render as month / week pickers." and, after it, a
second normative block following the one on primary keys.

A `uuid` field is the record's stable external identity, so its value is the server's to issue: a
client never chooses it.

> **Normative.**
> On a create through the generated application's API, on every surface that accepts one, a
> conforming generator MUST ignore a value the client sends for a non-key `uuid` field, without
> refusing the create, and MUST store a freshly drawn UUID. An update MUST keep the stored value. A writer inside the application
> that creates the record - a create-from, a posting, an import, an inbound message, hand-written
> code - MUST receive a drawn UUID unless its declaration maps a value into the field, and data
> loaded below the application, such as seed data or a migration, MUST keep the UUID it carries.
> The field MUST need no declaration beyond `type: uuid`.
