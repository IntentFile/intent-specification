# An action may be the record's filled button when it applies - `primary: true`

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/157, https://github.com/eclipse-dirigible/dirigible/issues/8119, https://github.com/eclipse-dirigible/dirigible/issues/8118
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8120

## The problem

A claim in DRAFT has one thing to do next - Submit - yet the filled button of its details sheet and
of its record page is Edit, and Submit sits in the overflow menu beside every other action. The model
cannot say which of a record's actions is the one the record is waiting for. 1.11 has no such key:
the record's filled button is always Edit, or a pending process task.

A second gap sits next to it. A `related:` register lists the records that reference this one and
offers only Open on each row, so the action a referencing record is waiting for - retry a failed
escalation, from the ticket it escalates - is two pages away.

## The proposed shape

```yaml
actions:
  - name: SubmitClaim
    forEntity: Claim
    label: Submit
    fromStatus: [DRAFT]           # the status guard of proposal 0095
    primary: true                 # when it applies, it is the filled button
    call: { path: /services/claims/{id}/submissions }

transitions:
  - name: RetryEscalation
    forEntity: Escalation
    from: [FAILED]
    setStatus: PENDING
    primary: true                 # also on a transitions: entry and a generates: entry
```

## Expected behaviour

- An action that applies to the record - its status guard holds and the caller holds its roles -
  takes the filled slot of the list's details sheet and of the record page, and leaves the menu it
  would otherwise sit in.
- Edit then moves into the sheet's menu, or becomes the page's outlined button, as it does for a
  pending task.
- The filled slot goes, in order, to a pending process task, then to a primary action that applies,
  then to Edit or Preview.
- Several primary actions of one entity are fine when they apply in different statuses; the first that
  applies wins.
- Each row of a `related:` register offers its record's own actions, decided by that row's status
  and the caller's roles, beside Open. The register stays read-only: no edit, delete or create.
  After an action runs, the register reads its rows again. A register whose source belongs to
  another model offers no actions, since that model's actions belong to its own application.

## Edge rules

- `primary` is accepted on an entity-scope `actions:` entry, on a `generates:` entry and on a
  `transitions:` entry, and refused on `scope: page`.
- Default `false`. A model without `primary` generates byte-identical descriptors.

## Prior art / workarounds

None within the format: the action could only sit in the menu, and the reader had to know it was
the one to choose. Reference implementation: https://github.com/eclipse-dirigible/dirigible/pull/8120.

## Specification text

**Anchor:** Processes & forms > actions - custom buttons, appended after the text proposal 0095
adds; a sentence appended to Declarative glue > transitions - guarded status flips and to Declarative
glue > generates - create-from; and a sentence appended to Relations & multi-model > related - list
the records that reference this entity.

*In actions - custom buttons:*

Most records wait for one thing at a time: a draft claim for its submission, a failed delivery for
its retry. `primary: true` names that action, so the record offers it first:

```yaml
actions:
  - name: SubmitClaim
    forEntity: Claim
    label: Submit
    fromStatus: [DRAFT]
    primary: true
    call: { path: /services/claims/{id}/submissions }
```

While the action applies to the record it is the record's filled button - on the details sheet the
list opens and on the record page - and Edit steps aside, into the sheet's menu or to the page's
outlined button, as it does for a pending task. Where it does not apply, the record offers Edit
first as before.

> **Normative.**
> `primary: true` MAY be declared on an entity-scope action, on a `generates:` entry and on a
> `transitions:` entry, and MUST be reported as an authoring error on an action with `scope: page`.
> While a primary action applies to a record - its status guard holds and the caller holds its
> roles - a conforming generator MUST render it as the filled button of the record's details sheet
> and of its record page and MUST NOT also list it in the menu it fills; Edit, or Preview, MUST then
> be offered in the sheet's menu and as the page's outlined button. The filled slot MUST go to a
> pending process task first, then to a primary action that applies, then to Edit or Preview. Where
> several primary actions of an entity apply to one record, the first that applies MUST win. A model
> without `primary` MUST generate as it did before.

*In transitions - guarded status flips, and in generates - create-from:*

An entry MAY declare [`primary: true`](#actions--custom-buttons), with the effect it has on an
action: while the entry applies to the record, it is the record's filled button.

*In related, after the normative block:*

> **Normative.**
> Each row of a same-model register MUST offer the referencing record's own per-record actions that
> apply to that row, decided by the row's status and the caller's roles, and the register MUST read
> its rows again once one has run. This does not relax the register's
> read-only rule: the actions are the source entity's declared operations, guarded by their own
> endpoints. A cross-model register MUST NOT offer actions.

## DSL index

| Construct | What it does |
| --- | --- |
| [`actions[].primary`](#actions--custom-buttons) | the action - or a `transitions:` / `generates:` entry - is the record's filled button while it applies; a pending task still comes first |
