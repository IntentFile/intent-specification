# A notify recipient or placeholder walks a chain of to-one relations

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/133, https://github.com/eclipse-dirigible/dirigible/issues/8030
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8032

## The problem

A notify block resolves its recipient and its `{placeholder}`s against the record it is about. 1.11
admits "a literal, a direct field, or a one-hop `relation.field`", and "Multi-hop paths (`a.b.c`) are
rejected with a clear message". The record a message is about often reaches what its reader needs
only through a record it belongs to: a renewal belongs to a subscription, and the mail about it
names the subscription's customer and plan.

Today a model has to **copy** those relations onto the renewal so the walk becomes one hop. That is a
second source of the same fact: nothing keeps the copy equal to the subscription's, and every writer
of the renewal must set it. And 1.11 says nothing of a multi-dot `{placeholder}`: in the reference
implementation it was not recognised as a path at all and went out as literal text, with no warning.

The format already walks such chains elsewhere: an assignee resolver path follows any number of
to-one relations, validated hop by hop when the file is read (Processes & forms > Task assignment:
"Every segment of `path` is a **to-one relation** ...").

## The proposed shape

```yaml
entities:
  - name: Subscription
    fields:
      - { name: id, type: integer, primaryKey: true, generated: true }
    relations:
      - { name: customer, kind: manyToOne, to: Customer, required: true }
      - { name: plan,     kind: manyToOne, to: Plan,     required: true }
  - name: Renewal
    fields:
      - { name: id,          type: integer, primaryKey: true, generated: true }
      - { name: requestedBy, type: string }
    relations:
      - { name: subscription, kind: manyToOne, to: Subscription, required: true }

notifications:
  - name: renewalRequested
    event: { onCreate: Renewal }
    to: requestedBy
    cc: subscription.customer.email                                  # a chain, in a recipient
    subject: "Your {subscription.plan.name} renewal for {subscription.customer.name}"
    body: "Open it at {subscription.plan.portalUrl}"                 # subscription is read once
```

## Expected behaviour

A notify path - a recipient and every `{placeholder}` of `subject`, `body` and `html` - may walk a
**chain of to-one relations** and end at a field of the last target. It does so at every notify call
site (`notifications[]`, `schedules[].notify`, `transitions[].notify`, a `serviceTask`'s
`args.notify`), against the record that site's notify is about (the row, inside a `forEach`).

Each related record is read once per message, however many paths share its prefix: in the example
the subscription is read once for its customer and its plan. A null relation anywhere on the chain
renders the value empty, exactly as a null one-hop relation does, and a recipient that so resolves
to no address is the existing no-recipient no-op.

## Edge rules

- Every segment but the last MUST name a to-one relation: the first of the record, each further one
  of the previous segment's target. The last MUST name a field of the last target.
- A **cross-model** relation may only be the **last** relation of a chain: its target is a
  projection that carries properties but no relations - the assignee resolver path's rule. Its field
  is checked against the owning model.
- A path of two or more relations that does not resolve MUST be rejected when the file is read,
  naming the block, the path and the failing segment. It MUST NOT be sent as literal text.
- A to-many segment MUST be rejected: a walk never fans out.
- Inside a `forEach` the walk starts at the row, as every bare path does. The reserved link
  placeholders, the `record.` scope of a fan-out and the `escalation.` scope of a schedule keep their
  own rules: one field, no walk.
- Direct fields and one-hop paths behave as before, including how an unresolvable one-hop path is
  treated today.
- The other one-hop sites (a `payload:` value, a `map:` source, a file-name pattern, a report
  `bind:`, an entity `label:` token) are unchanged.

## Prior art / workarounds

Copying the relations onto the record the message is about, or a hand-written sender. The chain
grammar is the assignee resolver path's. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8032.

## Specification text

**Anchor:** Declarative glue > notifications, the paragraph after the first example (1.11: "`to`
resolves a **literal** address, ... Multi-hop paths (`a.b.c`) are rejected with a clear message.").
Amended as follows, with consequential amendments listed at the end.

In that paragraph, "a **one-hop `relation.field`** of a to-one relation" (for `to`) and "Every
`{placeholder}` resolves a literal, a direct field, or a one-hop `relation.field`" become "a
**relation path**", and the last sentence ("Multi-hop paths (`a.b.c`) are rejected with a clear
message.") is replaced by:

A **relation path** is a chain of to-one relations ending at a field of the last target:
`customer.name`, `subscription.customer.name`. Every segment but the last names a to-one relation -
the first of the record the block is about, each further one of the previous target - and a
cross-model relation may only be the last relation of the chain, since its target is a projection
that carries properties but no relations. A null relation on the chain renders the value empty.

```yaml
notify:
  cc: subscription.customer.email
  subject: "Your {subscription.plan.name} renewal for {subscription.customer.name}"
```

> **Normative.** A conforming generator MUST validate every segment of a relation path of two or
> more relations when the file is read, and MUST reject a path that does not resolve, naming the
> block, the path and the failing segment; it MUST NOT send an unresolved placeholder as literal
> text. A to-many segment MUST be rejected: a walk never fans out. Each related record SHOULD be read
> at most once per message, however many paths share its prefix. Inside a `forEach` a relation path
> walks from the row. The reserved link placeholders and the `record.` and `escalation.` scopes are
> not relation paths and keep their own rules. A direct field and a one-hop path MUST behave as
> before.

Consequential amendments:

- The comments of the notify examples ("a literal, a direct field, or a one-hop relation.field",
  "literal / direct field / one-hop relation.field", "{field} and {relation.field} interpolation")
  read "relation path" in place of the one-hop wording.
- Wherever the format refers to "the forms `to` takes", the relation path is one of them.
- Declarative glue > integrations > payload - the declared envelope: the sentence "The value forms
  are the ones `notify` already resolves, deliberately borrowed rather than invented: a **literal**,
  a **direct field**, or a **one-hop `relation.field`** ..." keeps its one-hop rule and reads "the
  ones `notify` resolves, limited to a single hop". A `map:` source, a file-name pattern, a report
  `bind:` and an entity `label:` token stay one hop.

## DSL index

| Construct | What it does |
| --- | --- |
| [notify relation path](#notifications) | a notify recipient or `{placeholder}` that walks a chain of to-one relations to a field of the last target (`{subscription.customer.name}`), each related record read once per message; validated hop by hop when the file is read |
