# What a copy does not carry over

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7358
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7360

## The problem

`duplicable: true` is a bare boolean. The *Duplicate* action it adds clones the source header - minus
the values no user authored (identity, audit columns, status, document number, read-only and
aggregate fields) - and posts the rest through the normal create path. **Every ordinary user field
therefore rides along, including the ones a business rule says must be fresh.**

On a sales invoice that is the three dates. The copy carries the source's `date`, `due` and
`taxEventDate`, so "same invoice as last month" opens as a draft dated last month, due last month,
with last month's tax event:

```
SalesInvoice SI00000007   date 2026-08-01   due 2026-08-15   taxEventDate 2026-08-01
Duplicate ->  new draft   date 2026-08-01   due 2026-08-15   taxEventDate 2026-08-01
```

The author has to notice and fix three dates by hand, and an unnoticed one is a legally wrong
document - a tax-event date is mandatory on an invoice in several jurisdictions.

The model cannot say otherwise on its own side. A create-time rule
(`calculatedActionOnCreate` / `defaultValue`) fills an **empty** value and respects a present one -
the right contract for a hand-made invoice, and precisely what makes the copied value stick. `date`
carries no rule at all, and the format has no `defaultValue: now` for a date field.

## The proposed shape

`duplicable: true` stays the shorthand; the key additionally accepts a mapping.

```yaml
- name: SalesInvoice
  duplicable:
    defaults: { date: now }        # constants written into the clone
    reset: [due, taxEventDate]     # dropped, so the entity's own create-time rule refills them
  fields:
    - { name: date,         type: date, required: true }
    - { name: due,          type: date, calculatedActionOnCreate: custom.DueDate }
    - { name: taxEventDate, type: date, calculatedActionOnCreate: custom.TaxEventDate }
```

Two keywords rather than `defaults: { due: null }`: YAML `null` as a "default" is easy to misread,
and `reset` says what actually happens - the value is handed back to the entity's create-time rule
instead of being assigned. They compose: `reset` for a field that HAS such a rule, `defaults` for a
field that does not.

## Expected behaviour

A conforming generator MUST build the cloned header in this order:

1. the built-in drops, unchanged and not authorable: the primary key, the audit columns, the
   `function: EntityStatus` relation, the `number:` field, and every `readOnly` or `aggregate`
   property;
2. every name in `reset:`, deleted from the clone;
3. every entry in `defaults:`, assigned after the resets;
4. everything else copied from the source, as before.

`now` is **today in the target property's own shape** - `YYYY-MM-DD` for a `date` field, `YYYY-MM`
for a `month` field, `YYYY-Www` for a `week` field - the same token and the same shape rule
`generates.defaults` carries. It MUST be resolved against the LOCAL calendar of the acting user, not
UTC: east of Greenwich a UTC-derived date is yesterday for the last hours of every day, so the very
document the construct exists to date correctly would be dated wrong.

Any other `defaults` value is a literal coerced to the property's declared type - a number for a
numeric property, a boolean for a boolean, the string as authored otherwise, and the raw foreign key
for a to-one relation. The property's declared type decides, not the value's shape: a string field
holding `"01"` stays the string it was authored as.

An entity declaring `duplicable: true` and nothing else MUST generate exactly what it generated
before this proposal.

## Edge rules

- Both keys name **the entity's own fields and to-one relations**; a `relation.field` path is
  rejected.
- A name that is neither a field nor a to-one relation of the entity is rejected.
- A name that is one of the built-in drops is rejected rather than accepted and ignored: it is
  already dropped, and naming it would let an author believe they control something the Duplicate
  decided long before reading this block.
- The same name in `reset` and `defaults` is rejected - a contradiction.
- `now` on a property that is not a `date`, `month` or `week` is rejected.
- A `reset` of a **required** value the create cannot fill on its own is rejected: a field with
  neither a `defaultValue` nor a create-time rule, or a to-one relation that declares no `init:`. The
  clone would be refused by the server on every attempt, which is a mistake at authoring time and not
  a decision.
- A `duplicable` value that is neither `true`, `false` nor a mapping is rejected.
- The object form on an entity that is not a document is accepted and ignored exactly as
  `duplicable: true` is - the key has never been an error there.
- The copy's line items are unaffected: they are cloned as before, repointed at the new header.

## Prior art / workarounds

Today the author fixes the dates by hand on every copy, or the module gives up on `duplicable` and
asks users to key the document again. The obvious implicit alternative - dropping every
`calculatedActionOnCreate` field on duplicate, with no authoring change - was considered and
rejected as a default: it would also recompute fields the source deliberately overrode (a currency,
a bank account), and the business rule differs per document (an invoice resets its due date, a
quotation would reset `validUntil`), which is exactly what the intent should state rather than
guess.

## Specification text

**Anchor:** Entities > Entity-level attributes (immediately after the *Control order* subsection)

#### What a copy does not carry over

`duplicable: true` adds a *Duplicate* button to a document entity. It clones the current document -
header plus line items - into a new draft and opens it, through the normal create path, so the
document number, the initial status, the audit columns and every calculated or aggregate field are
reassigned. Everything else is copied.

Copying everything else is wrong for the fields a business rule says must be fresh - the date of an
invoice, its due date, its tax-event date. A create-time rule cannot correct them, because such a
rule fills an empty value and respects a present one. The object form of `duplicable` states which
fields the copy does not carry over:

```yaml
- name: SalesInvoice
  duplicable:
    defaults: { date: now }        # constants written into the clone
    reset: [due, taxEventDate]     # dropped, so the entity's own create-time rule refills them
```

`reset:` is for a field that has a create-time rule (`calculatedActionOnCreate`, `defaultValue`):
each name is dropped from the clone, so the create fills it exactly as it would on a hand-made
document. `defaults:` is for a field that has none: each entry is assigned into the clone after the
resets. `now` is today in the field's own shape - `YYYY-MM-DD` for a `date` field, `YYYY-MM` for a
`month` field, `YYYY-Www` for a `week` field - and any other value is a literal coerced to the
property's declared type. Both keys name the entity's own fields and to-one relations.

> **Normative.**
> The clone is built in this order: the built-in drops (primary key, audit columns, the
> `function: EntityStatus` relation, the `number:` field, every `readOnly` and `aggregate`
> property), then the `reset:` names, then the `defaults:` assignments, then every remaining
> property copied from the source. `now` resolves against the acting user's local calendar, never
> UTC. `duplicable: true` alone generates as it did before this form existed.

> **Normative.**
> The following are invalid: a `reset` or `defaults` name that is not a field or a to-one relation
> of the entity; a name that is one of the built-in drops; the same name in both keys; `now` on a
> property that is not a `date`, `month` or `week`; a `reset` of a required field with neither a
> `defaultValue` nor a create-time rule; and a `duplicable` value that is neither `true`, `false`
> nor a mapping.

## DSL index

| Construct | What it does |
| --- | --- |
| `duplicable: { defaults, reset }` | says which fields a copy assigns and which it hands back to the entity's create-time rules |
