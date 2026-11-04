# An action or transition may be destructive - `destructive: true` - a red run button, placed beside Delete

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/165, https://github.com/eclipse-dirigible/dirigible/issues/8364
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8381

## The problem

An action that throws data away - purge a closed ticket, wipe a mailbox's archive - looks exactly
like one that exports it: the same button colour, the same place in the record's menu, the same
confirm dialog. Delete is set apart in every record menu; an action that is just as final is not,
and 1.11 has no key that would say so.

## The proposed shape

```yaml
actions:
  - name: PurgeArchive
    forEntity: Mailbox
    label: Purge archive
    fromStatus: [CLOSED]                                   # proposal 0095
    confirm: "Delete every archived message of {address}?" # proposal 0095
    destructive: true
    call: { method: DELETE, path: /services/mailboxes/{id}/archive }

transitions:
  - name: DiscardDraft
    forEntity: Claim
    from: [DRAFT]
    setStatus: DISCARDED
    destructive: true
```

## Expected behaviour

- The run button of the action's confirm or prompt dialog renders in the negative (destructive)
  variant; its caption is unchanged. A non-destructive action's dialog keeps its usual button.
- In every record menu - a list row, the details sheet, the record page, a composition panel row, a
  `related:` register row, a document page - the action leaves the group of actions and joins the
  destructive group: after the separator, directly above Delete, in the negative variant.
- Where a menu has no Delete, the destructive actions are its last group, after a separator. The
  group shows when it holds a destructive action or Delete; a composition panel row whose only
  action is destructive still gets its menu.
- Nothing changes server side: the action calls, guards and refuses exactly as it would without the
  flag.

## Edge rules

- Accepted on an entity-scope `actions:` entry (a `call:` or a `page:` action) and on a
  `transitions:` entry. Default `false`.
- Refused together with `primary: true` (proposal 0096): a destructive action is never the record's
  filled button.
- Refused on `scope: page`: the destructive group belongs to a record's menu.
- Both refusals name the action. A model without `destructive` generates byte-identical
  descriptors.

## Prior art / workarounds

None within the format: the only place to warn was the label and the confirm text, on a button
that looked like every other. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8381.

## Specification text

**Anchor:** Processes & forms > actions - custom buttons, appended after the `primary` text of
proposal 0096; and a sentence appended to Declarative glue > transitions - guarded status flips.

*In actions - custom buttons:*

An action that destroys data should look like Delete, not like Export. `destructive: true` says so:

```yaml
actions:
  - name: PurgeArchive
    forEntity: Mailbox
    label: Purge archive
    fromStatus: [CLOSED]
    confirm: "Delete every archived message of {address}?"
    destructive: true
    call: { method: DELETE, path: /services/mailboxes/{id}/archive }
```

The action's dialog runs from a destructive button, and in every menu of the record the action sits
with Delete rather than with the other actions. It changes how the action is offered, never what it
does.

> **Normative.**
> `destructive: true` MAY be declared on an entity-scope action and on a `transitions:` entry; the
> default is `false`. For a destructive action a conforming generator MUST render the run button of
> its confirm or prompt dialog in the destructive variant, keeping its caption, and MUST place the
> action in every menu that offers it to a record - a list row, the details sheet, the record page,
> a composition panel row, a `related:` register row and a document page - in the destructive group:
> after a separator and directly above Delete, or, in a menu without Delete, as the last group after
> a separator. The group MUST be shown when it holds a destructive action or Delete, and a menu
> whose only offered action is destructive MUST still be rendered. The flag MUST NOT change the
> action's server-side behaviour. `destructive: true` together with `primary: true`, and on an action
> with `scope: page`, MUST be reported as authoring errors naming the action. A model without
> `destructive` MUST generate as it did before.

*In transitions - guarded status flips:*

A transition MAY declare [`destructive: true`](#actions--custom-buttons), with the effect it has on
an action - a void or a discard that cannot be taken back sits beside Delete.

## DSL index

| Construct | What it does |
| --- | --- |
| [`actions[].destructive`](#actions--custom-buttons) | the action - or a `transitions:` entry - runs from a destructive button and sits with Delete in every record menu; presentation only |
