# Entity-level unique: a business key over more than one field

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/27
- **Reference implementation:** https://github.com/eclipse-dirigible/dirigible/issues/6763

## Why

`unique` is a **field** attribute, so a business key spanning more than one column has no expression:
one row per `(tenant, application)`, one assignment per `(tenant, user)`, one price per
`(product, priceList, validFrom)`. `checks` covers row conditions and aggregate guards, not
uniqueness.

The rule that defines what a row *is* then has nowhere to live in the model. In practice it ends up
in one of two places, and both are worse than a declaration:

- a **read-then-write in hand-written code** — a race by construction, and one that every writer has
  to repeat: the form, an import, an arriving message, a scheduled creation;
- a **constraint added to the generated schema by hand** — which the next generation knows nothing
  about, so the rule survives exactly until someone regenerates.

Neither is visible in the intent, so the single most important sentence about the entity — what
makes two rows the same row — is the one sentence the model does not say.

## What this adds

`unique` on an **entity**, one entry per key:

```yaml
entities:
  - name: TenantApplication
    unique:
      - { fields: [tenant, application], message: "This application is already provisioned for the tenant" }
    fields:
      - { name: id,   type: integer, primaryKey: true, generated: true }
      - { name: plan, type: string }
    relations:
      - { name: tenant,      kind: manyToOne, to: Tenant, required: true }
      - { name: application, kind: manyToOne, to: Application, required: true }
```

`fields` names fields **or to-one relations** — a relation contributes its foreign-key column, which
is what a pair like `(tenant, application)` means. The columns are constrained in the declared
order, and a colliding write is answered as a conflict carrying `message`.

## The normative half

The key is enforced **by the data store**, not by the application: that is the whole point of
declaring it, because it is the only place the rule holds for every writer at once, including the
ones the model does not know about.

A colliding write must be reported as a **conflict** distinguishable from a generic failure, and the
authored `message` must be what the caller is told — a duplicate is an outcome a user acts on, not
an error a user reports.

What is rejected at authoring time: a name that is neither a field nor a to-one relation of the same
entity (a to-many has no column on this side to constrain); a single-name key, which names the field
attribute it duplicates, because two ways to say one thing is how the two drift apart; a repeated
name inside one key; the same key declared twice.

Appendix A gains its row.

## Notes

Deliberately **not** part of this proposal:

- **Partial or filtered keys** (unique among non-cancelled rows). That is a different construct with
  a different failure mode, and the data stores disagree on whether they can express it at all.
- **Retrofitting an existing table.** A key declared on an entity whose table already exists carries
  the same caveat as any other schema change; a conforming implementation is not required to alter a
  table that is already there.
- **A cross-model relation in a key.** A conforming implementation may reject it: whether the
  consumer holds a real column for a cross-model reference is an implementation matter the format
  does not settle, and the conservative reading is the one that can be relaxed later without
  breaking authored models.

## Specification text

The prose below is what a release folds into the next version document, at the anchor given. It was
written against `versions/1.2.md` and is carried here unchanged.

**Anchor:** Entities > Field and relation attributes (after `unique`)

#### unique — a business key over more than one field

`unique: true` on a field constrains one column. When what makes a row unique spans several,
declare it on the entity instead:

```yaml
entities:
  - name: TenantApplication
    unique:
      - { fields: [tenant, application], message: "This application is already provisioned for the tenant" }
```

`fields` names fields or to-one relations of the same entity, in the order the key constrains them;
a to-one relation contributes its foreign-key column. `message` is what a caller is told when a
write collides; omitted, an implementation derives one from the names.

> **Normative.**
> The key MUST be enforced by the data store, so that it holds for every writer — a form, an import,
> an arriving message, a scheduled creation — and not only for the ones that route through the
> application.
> A colliding write MUST be reported as a conflict that a caller can distinguish from a generic
> failure, carrying the authored `message` when one was given.
> Every name MUST resolve to a field or a to-one relation of the same entity; a to-many MUST be
> rejected, having no column on this side to constrain.
> A key naming a single field MUST be rejected, naming the field-level `unique` it duplicates.
> A name repeated within one key, and a key declared twice on one entity, MUST be rejected.
> An implementation is NOT required to add the constraint to a table that already exists.

**Anchor:** Appendix A: DSL index

| [`entities.unique`](#unique--a-business-key-over-more-than-one-field) | a business key spanning more than one field or to-one relation |
