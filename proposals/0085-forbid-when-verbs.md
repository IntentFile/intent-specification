# A `forbidWhen` scoped by `verbs` refuses only the writes it names

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/146, https://github.com/eclipse-dirigible/dirigible/issues/7710
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7711
- **Amends:** [0047 - checks: forbidWhen](0047-checks-forbid-when.md), [0069 - a `forbidWhen` refuses the delete](0069-forbid-when-delete-and-presence.md)

## The problem

Since [0069](0069-forbid-when-delete-and-presence.md) a `forbidWhen` "refuses a create, update or
delete of the row while its condition holds". That is right for "no line may change on a sent
quotation". It is wrong for a rule about **adding** only. "No allocation onto a PAID invoice" now
also makes an allocation booked onto the wrong invoice impossible to remove; and with the default
`restrict` of `whenTargetDeleted` on a referencing to-one, it blocks the delete of the payment the
allocation belongs to. The opposite rule - a row written freely and refused only its removal - is
not expressible either. 1.11 gives `kind: forbidWhen` no way to say which writes a rule covers.

## The proposed shape

```yaml
- name: SalesInvoiceCustomerPayment
  checks:
    # adding and changing are refused on a paid invoice; removing a wrong allocation is not
    - { kind: forbidWhen, when: "SalesInvoice.Status == PAID", verbs: [create, update],
        message: "Cannot add a payment to a fully paid invoice" }
- name: Note
  checks:
    # written freely; only its removal is refused while it is locked
    - { kind: forbidWhen, when: "locked == true", verbs: [delete],
        message: "A locked note cannot be removed" }
```

## Expected behaviour

- `verbs:` is an optional list over `create`, `update` and `delete`. Without it a rule covers all
  three, exactly as in 1.11.
- A rule scoped to `[create, update]` refuses the row's create and update while the condition holds,
  with its message (400 at the endpoint), and accepts its delete.
- A rule scoped to `[delete]` lets the row be created and updated freely and refuses only its
  removal while the condition holds against the stored row.
- On the master-detail panel that shows the row as a child of its master, only the affordances the
  rule covers are hidden: Add for `create`, row edit for `update`, row delete for `delete`.

## Edge rules

Refused when the intent is read:

- an empty `verbs` list;
- an unknown verb;
- `verbs` on any check kind other than `forbidWhen`;
- `verbs` together with `severity: warn` - a warning never reaches the delete;
- `create` without `update`, or `update` without `create`: the two are one pair, because one
  validation of the written row serves both verbs on every write path;
- the key spelled `on:`, which a YAML reader takes for the boolean `true`; the report names `verbs`.

A file that declares no `verbs` generates byte-identically.

## Prior art / workarounds

Before the scope, the rule covered every verb, and a module that needed the removal had to work
around it (a cascade around the guarded child). Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/7711.

## Specification text

**Anchor:** Entities & fields > checks > kind: forbidWhen - a write refused while a condition holds,
appended after the delete paragraph of [0069](0069-forbid-when-delete-and-presence.md). Amends 0047
and 0069: the first sentence of the section becomes "refuses a create, update or delete of the row
- or only the ones its `verbs` names - while its condition holds".

A rule may name the writes it covers. `verbs` lists them, over `create`, `update` and `delete`;
without it a rule covers all three. A rule about adding leaves the removal open, and a rule about
removal leaves the row free to be written:

```yaml
- name: SalesInvoiceCustomerPayment
  checks:
    - { kind: forbidWhen, when: "SalesInvoice.Status == PAID", verbs: [create, update],
        message: "Cannot add a payment to a fully paid invoice" }
- name: Note
  checks:
    - { kind: forbidWhen, when: "locked == true", verbs: [delete],
        message: "A locked note cannot be removed" }
```

`create` and `update` are one pair: one validation of the written row serves both, so a rule names
both or neither.

> **Normative.**
> A `forbidWhen` without `verbs` MUST cover the create, the update and the delete of the row. With
> `verbs` it MUST refuse exactly the writes it lists while the condition holds, and MUST NOT refuse
> the others: a rule listing `create` and `update` MUST accept the delete, and a rule listing only
> `delete` MUST accept the create and the update. On the master-detail panel the generator MUST hide
> only the affordances of the listed verbs - the add affordance for `create`, the row edit for
> `update`, the row delete for `delete` - and the server refusal of the listed verbs stands
> underneath on every path. A generator MUST reject as an authoring error: an empty `verbs`; a verb
> other than `create`, `update` and `delete`; `verbs` on a check kind other than `forbidWhen`;
> `verbs` together with `severity: warn`; and a list naming one of `create` and `update` without the
> other. A file declaring no `verbs` MUST generate byte-identical output.

## DSL index

No new row; the `checks: kind: forbidWhen` row becomes "a create, update or delete refused while a
condition over the record or one hop holds, optionally scoped to the writes `verbs` names; `== null`
/ `!= null` test presence".
