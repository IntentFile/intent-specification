# Role-scoped field visibility (`visibleTo`)

- **Status:** released in [1.4](../versions/1.4.md)
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/6550

## The problem

A field is as visible as its entity. `sensitive: true` removes one from the **scoped** surfaces
(personal / partner), but nothing scopes a field on the surface everybody else uses, so the figures
an organisation is most careful about travel to every user who may read the record at all:

- a **salary** or **daily rate** on an employee, readable by whoever may open the employee list;
- a **cost price** or **margin** on a product or an order line, readable by every salesperson;
- a **credit limit** or **risk score** on a customer.

The only tool available today is to hide the control in the user interface, which is not a
protection at all: the value is in the REST response the page rendered from, and in the change
history, and in the CSV export. And moving the field to its own entity - the usual workaround -
splits a record that is conceptually one, for a reason that is not about the data.

## The proposed shape

An **allow-list** of roles on the field:

```yaml
permissions:
  - { role: Payroll }
  - { role: Administrator }

entities:
  - name: Employee
    fields:
      - { name: id,        type: integer, primaryKey: true, generated: true }
      - { name: name,      type: string,  required: true }
      - { name: dailyRate, type: decimal, visibleTo: [Payroll, Administrator] }
```

Holding **any one** of the listed roles is enough. Absent (the default), a field is visible to
every caller who may read the entity - nothing changes for existing files.

The inverse spelling (`hiddenFor: [Role]`) is deliberately **not** proposed: a deny-list fails
open. A role added to the application after the file was written would see the value, and a
misspelled role would expose it rather than hide it. An allow-list fails closed in both cases.

## Expected behaviour

A conforming generator MUST enforce the list where the data leaves the server, not in the
presentation layer:

- **Reads** - the property is absent (or null) from every response of every generated surface -
  the main one and the scoped (`personal` / `partner`) ones - unless the caller holds one of the
  roles. Owning the record, or being the partner it belongs to, does not grant a role.
- **Writes** - a create ignores the submitted value; an update keeps the stored one. Refusing the
  whole request is NOT required: the field is not the caller's to set, and the rest of the write is
  legitimate.
- **Change history** - where the format records a field-level trail, the entries of a restricted
  property are withheld from a caller who may not read it. They carry the same values.
- **Derived values** - a derived field fed by a restricted one (a roll-up, an aggregated master
  total, a keyed aggregate) inherits the same allow-list unless it declares its own. A sum of
  hidden figures is that figure one entity out.
- **Presentation** - the generated UI SHOULD leave the field's column / input out for a caller who
  cannot read it, and MUST derive that from the server (which fields are withheld from *this*
  caller), not from a role list evaluated in the client. The redaction on the wire is the
  enforcement; the hiding is courtesy.

## Edge rules

- Every listed role MUST be declared in the file's `permissions`. A role no permission grants would
  hide the field from everybody, which is a typo far more often than an intention, and produces
  nothing observable anywhere - it is rejected, naming the declared roles.
- An **empty** list (`visibleTo: []`) is rejected rather than treated as "no restriction": the two
  are indistinguishable to a typed parser, and the author who wrote the key meant the opposite of
  what dropping it produces.
- Rejected on the primary key, on the entity's `identity` field, and on a field carrying the
  document title: every response is addressed by the key, the scoped surfaces resolve the caller
  through the identity field, and the document title is the form's heading. Hiding any of them does
  not produce a restricted field, it produces a broken page.
- Rejected as a `label` token (own field or one-hop relation property). A label materialises a
  stored, ordinary display-name column that every reader of the entity gets - the same rule the
  format already applies to `sensitive` fields.
- A **report** over a restricted field is a WARNING, not a rejection: a report carries no
  field-level scoping, so it re-serves the figure to everyone its own roles admit. That is a
  legitimate thing to author (a payroll report over payroll data is the point), so the author is
  told which report re-exposes which field, and scopes the report accordingly.
- `sensitive` and `visibleTo` are independent and may appear together: the first is about a
  surface, the second about a role.
- Scoping applies per field. A relation (a foreign key) is out of scope for this proposal: hiding
  one hides a dropdown the record is filed under, which is a layout question rather than a
  visibility one.

## Prior art / workarounds

Splitting the restricted fields into a satellite entity with its own roles - which fragments one
record and leaves the roll-ups, print templates and imports to be re-plumbed - or hiding the
control in the UI and accepting that the API serves the value anyway. Entity-level read/write
roles exist in several model-driven platforms (including the reference implementation's own entity
modeler, which carries a per-property read and write role); what the format lacks is a way to say
it one altitude up, where the field is declared.
