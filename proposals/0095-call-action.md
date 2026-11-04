# An action may call the application's own endpoint

- **Status:** released in [1.12](../versions/1.12.md)
- **Issue:** https://github.com/IntentFile/intent-specification/issues/156, https://github.com/eclipse-dirigible/dirigible/issues/8090, https://github.com/eclipse-dirigible/dirigible/issues/8296, https://github.com/eclipse-dirigible/dirigible/issues/8323
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8092, https://github.com/eclipse-dirigible/dirigible/pull/8305, https://github.com/eclipse-dirigible/dirigible/pull/8326

## The problem

A hand-written endpoint that owns a business rule - one that guards its own state machine, is
idempotent, and answers a 409 written for people - cannot get a button in the generated screens.
1.11 describes `actions` as "Developer-defined buttons that open a custom page": an action has
`page:` and nothing else, so the application has to write a whole page whose only purpose is to send
one request. `generates` and `transitions` do send a request, but to the endpoints the generator
writes for them, which bypass the application's endpoint and the rules it holds.

Once an action can send that request, its dialog has to say what it acts on. It has to name the
record, caption its submit button with the verb, offer a choice of several values as one input whose
"at least one" is enforceable, open an input with a value and explain it. A dialog that changes a
record's child rows has to open with the rows the record already has ticked.

The construct is not the process step that 1.11 calls `call` (a service-task shape that runs a
script). The two share a word and nothing else: the step runs inside a process with no user in front
of it; the action is a button a user presses on a record.

## The proposed shape

```yaml
actions:
  - name: EscalateTicket
    forEntity: Ticket
    label: Escalate
    fromStatus: [OPEN, ON_HOLD]          # offered only in these statuses; the endpoint decides
    can: Ticket:escalate                 # offered only to the roles whose permissions list it
    prompt:
      - { name: level, type: integer, label: Level, required: true }
      - { name: note,  type: text,    label: Note }
    call:
      method: POST                       # POST (default) | PUT | DELETE
      path: /services/support/tickets/{id}/escalations
      body: { kind: MANUAL, level: "{level}", note: "{note}" }
    success: Escalation requested

  - name: RetryEscalation
    forEntity: Escalation
    label: Retry
    fromStatus: [FAILED]
    can: Escalation:retry
    confirm: Run this escalation again?
    call: { path: /services/support/escalations/{id}/retry }    # no body: nothing is sent

  - name: InviteMember
    forEntity: Team
    description: "Invite a person to {name}. They get an email with a link to the team's workspace."
    submit: Invite                       # the run button's caption
    prompt:
      - { name: email, type: email, label: Email, required: true, help: "name@company.com" }
      - name: roles
        type: multiselect
        label: Roles
        required: true                   # at least one
        default: [Member]
        options:
          - { value: Member, help: Uses the team's workspace }
          - { value: Lead,   help: Manages the team's members }
    call:
      path: /services/teams/teams/{id}/invitations
      body: { email: "{email}", roles: "{roles}" }

  - name: ChangeSkills
    forEntity: TeamMember                # list: [name, skills.skill]
    description: "Change the skills of {name}, now {skills.skill}."
    prompt:
      - { name: skills, type: multiselect, required: true, default: "{skills.skill}", options: [Java, SQL, Go] }
    call: { path: /services/teams/members/{id}/skills, body: { skills: "{skills}" } }
```

## Expected behaviour

- `call: { method, path, body }` makes the action send one request to the application's own
  endpoint. It is exclusive with `page:`. The button is offered from the record's menus and on the
  record page; `{id}` in the path is the clicked record's key.
- `prompt:` opens a small form before the request; `confirm:` asks a question before it. `success:`
  is the confirmation shown after a 2xx answer, by default "<label> completed". A 400 or 409 shows
  the endpoint's own message.
- `fromStatus:` offers the button only on a record standing in one of the listed statuses. `can:`
  names a permission token, and the button is offered only to callers holding a role whose
  `permissions` list it. Both are **advisory**: the generated application adds no check of its own
  on the server, and the endpoint decides.
- A `body` string that is exactly `"{name}"` sends that input typed by its control: a number, a
  boolean, `null` when left empty. `{name}` inside a longer string interpolates as text. Without
  `body`, the request carries none.
- `description:` (a line under the dialog's title) and `confirm:` may name properties of the record
  as `{property}`: a field, a to-one relation (shown by its display text) or the stored display name
  a `label:` generates. `submit:` captions the run button, by default "Run".
- An input may declare `default:` (a literal, or a record `{property}`) and `help:` (a line under the
  control). `type: multiselect` takes `options:`, each `{ value, label?, help? }`, a bare string being
  its own value and label. A multiselect is sent as an array in options order, `[]` when nothing is
  ticked; a required one needs at least one tick. A default fills only an input that is still empty
  when the dialog opens.
- A `default:`, `description:` or `confirm:` placeholder may name a collection path the entity's
  `list:` shows (`{skills.skill}`, or a bare `{skills}`). The dialog then opens with those texts
  ticked, and in prose they are joined with ", ". The option values must equal the texts.

## Edge rules

- `method` is `POST`, `PUT` or `DELETE`, default `POST`. `path` is absolute under the application's
  service root. `{id}` is the only placeholder a path may carry, and it requires `scope: entity`.
- Every `{name}` in `body` must be a declared prompt input. Prompt input types come from a closed
  list. `prompt` and `confirm` are exclusive.
- `fromStatus` requires the entity to declare a `function: EntityStatus` relation; its names resolve
  like every other status reference.
- A `can:` token that no role lists is reported, since the button would be offered to nobody.
- The keys that belong to the calling shape (`call`, `prompt`, `confirm`, `fromStatus`, `can`,
  `success`, `description`, `submit`) are refused on an action that opens a page. Any placeholder is
  refused on `scope: page`.
- Refused: a `{property}` the entity does not have; `options` on an input that is not a
  multiselect, and a multiselect without them; a blank or a repeated option value; a literal default
  that does not fit its input's type or names no option; a collection-path default on an input that
  is not a multiselect; and a collection path the entity's `list:` does not show.
- The collection-path placeholder relies on the collection path in `list:` that proposal 0102
  (`list-collections-and-audit`) adds; 1.11 `list:` refuses a to-many relation.
- That option values equal the collection's texts is the author's responsibility; it is not
  validated.
- An action declaring none of these keys generates exactly as before.

## Prior art / workarounds

A custom page whose script posted the one request, with its own dialog, toast and error handling;
or a `generates` / `transitions` entry, which reaches the generated endpoint and not the one holding
the rule. Several values were offered as one boolean input per value. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8092,
https://github.com/eclipse-dirigible/dirigible/pull/8305 and
https://github.com/eclipse-dirigible/dirigible/pull/8326.

## Specification text

**Anchor:** Processes & forms > actions - custom buttons. The section's opening sentence is amended
to "Developer-defined buttons that open a custom page or call the application's own endpoint - the
escape hatch when a workflow or a generated screen is not enough:", and the following is appended.

#### call - a button for the application's own endpoint

An action MAY declare `call:` instead of `page:`. The button then sends one request to an endpoint
the application itself provides - the shape for a hand-written endpoint that owns a business rule,
guards its own state machine and explains its own refusals, which `generates` and `transitions`
cannot reach because they send their request to the endpoints the generator writes for them:

```yaml
actions:
  - name: EscalateTicket
    forEntity: Ticket
    label: Escalate
    fromStatus: [OPEN, ON_HOLD]          # offered only in these statuses
    can: Ticket:escalate                 # offered only to roles whose permissions list it
    prompt:
      - { name: level, type: integer, label: Level, required: true }
      - { name: note,  type: text,    label: Note }
    call:
      method: POST                       # POST (default) | PUT | DELETE
      path: /services/support/tickets/{id}/escalations
      body: { kind: MANUAL, level: "{level}", note: "{note}" }
    success: Escalation requested
  - name: RetryEscalation
    forEntity: Escalation
    label: Retry
    fromStatus: [FAILED]
    confirm: Run this escalation again?
    call: { path: /services/support/escalations/{id}/retry }
```

The action is unrelated to the `call` service-task shape of a process: that step runs inside a
process instance with nobody in front of it, while a `call:` action is a button a user presses on a
record.

- `method` is `POST`, `PUT` or `DELETE` (default `POST`); `path` is an absolute path under the
  application's service root, in which `{id}` - the only placeholder it may carry - is the clicked
  record's key.
- `body` is the request's JSON body. A string that is exactly `"{name}"` sends that prompt input
  typed by its control (a number, a boolean, `null` when left empty); `{name}` inside a longer
  string interpolates as text. Without `body` the request carries none.
- `prompt:` is a small form shown before the request, one entry per input
  (`{ name, type, label?, required?, default?, help? }`); `confirm:` is a question asked before it
  instead. `success:` is the confirmation shown after a 2xx answer.
- `fromStatus:` lists the statuses the record must stand in for the button to be offered, by seeded
  name or id as every [status reference](#status-references--name-not-number) is; `can:` names a
  [permission](#permissions) token, and the button is offered only to callers holding a role whose
  `can` list carries it.

> **Normative.**
> A `call:` action MUST be exclusive with `page:`, and `call`, `prompt`, `confirm`, `fromStatus`,
> `can`, `success`, `description` and `submit` MUST be reported as authoring errors on an action
> that opens a page. `method` MUST be `POST`, `PUT` or `DELETE`; `path` MUST be absolute under the
> application's service root; `{id}` MUST be the only placeholder in the path and MUST be declared
> only on an action of `scope: entity`. Every `{name}` in `body` MUST name a declared prompt input;
> an input's `type` MUST come from the implementation's closed list of input types; `prompt` and
> `confirm` MUST NOT both be declared. `fromStatus` MUST be refused on an entity declaring no
> `function: EntityStatus` relation. A `can:` token that no role's `permissions` list carries MUST
> be reported, since the button would be offered to nobody.
>
> `fromStatus` and `can` decide only where the button is offered. A conforming generator MUST NOT
> add a server-side check for either: the endpoint the action calls is the enforcement, and its
> refusal - a 400 or a 409 - MUST be shown to the user with the endpoint's own message. A required
> prompt input MUST be enforced in the dialog before any request is sent. After a 2xx answer the
> user MUST be shown `success`, or "<label> completed" when none is declared. An action declaring
> none of these keys MUST generate exactly as before.

#### The dialog says what it acts on

A calling action's dialog MAY name the record it acts on, caption its run button, and offer, fill
and explain its inputs:

```yaml
actions:
  - name: InviteMember
    forEntity: Team
    description: "Invite a person to {name}. They get an email with a link to the team's workspace."
    submit: Invite
    prompt:
      - { name: email, type: email, label: Email, required: true, help: "name@company.com" }
      - name: roles
        type: multiselect
        label: Roles
        required: true                   # at least one
        default: [Member]
        options:
          - { value: Member, help: Uses the team's workspace }
          - { value: Lead,   help: Manages the team's members }
    call:
      path: /services/teams/teams/{id}/invitations
      body: { email: "{email}", roles: "{roles}" }
```

- `description:` is shown under the dialog's title, and `submit:` captions its run button (default
  "Run"). `description:` and `confirm:` may name properties of the clicked record as `{property}`:
  a field, a to-one relation, shown by its display text, or the stored display name a
  [`label:`](#label--a-stored-display-name) generates. A property the record holds no value in
  renders empty.
- `help:` is a line shown under an input's control.
- `type: multiselect` is one input offering several values. Its `options:` are
  `{ value, label?, help? }`, a bare string being its own value and label. It is sent as an array in
  options order, `[]` when nothing is ticked; a required multiselect needs at least one tick.
- `default:` opens an input with a value: a literal of the input's type, or a record `{property}`. A
  default fills only an input that is still empty when the dialog opens.

A dialog that changes a record's child rows opens with the rows the record already has. A
`default:`, `description:` or `confirm:` placeholder may name a collection path that the entity's
`list:` shows - `<collection>.<property>` or a bare `<collection>`:

```yaml
actions:
  - name: ChangeSkills
    forEntity: TeamMember                # list: [name, skills.skill]
    description: "Change the skills of {name}, now {skills.skill}."
    prompt:
      - { name: skills, type: multiselect, required: true, default: "{skills.skill}", options: [Java, SQL, Go] }
    call: { path: /services/teams/members/{id}/skills, body: { skills: "{skills}" } }
```

The multiselect then opens with the record's texts ticked, and in prose the texts are joined with
", ". The texts are matched against the option values, so the options are written as the texts the
collection holds.

> **Normative.**
> Every `{property}` in `description`, `confirm` or a record `default` MUST resolve to a field, a
> to-one relation or the stored display name of `forEntity`, or to a collection path its `list:`
> shows; an unknown name MUST be reported as an authoring error naming the action and the key, and
> any placeholder on an action of `scope: page` MUST be refused. `options` MUST be refused on an
> input that is not a multiselect, and a multiselect without them MUST be refused, as MUST a blank
> or a repeated option value. A literal default that does not fit its input's type, or that names
> no option of a multiselect, MUST be refused. A collection-path default MUST be refused on an input
> that is not a multiselect. A multiselect MUST be sent as an array of the ticked values in options
> order, and as `[]` when none is ticked; a required multiselect with no tick MUST be refused in the
> dialog before any request is sent. A default MUST NOT overwrite an input that already holds a
> value when the dialog opens. An action declaring none of `description`, `submit`, `help`,
> `default` or `options` MUST generate exactly as before.

## DSL index

The `actions` row becomes "developer-defined buttons opening custom pages or calling the
application's own endpoint", and a new row follows it:

| Construct | What it does |
| --- | --- |
| [`actions[].call`](#call---a-button-for-the-applications-own-endpoint) | a button that sends one request to the application's own endpoint, with a prompt or confirm dialog that names the record, advisory `fromStatus` / `can`, typed inputs including `multiselect`, defaults and help |
