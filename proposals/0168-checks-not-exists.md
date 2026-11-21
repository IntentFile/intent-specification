# A status move refused while a matching row of another entity exists - `checks: kind: notExists`

- **Status:** draft
- **Issues:** https://github.com/eclipse-dirigible/dirigible/issues/8002,
  https://github.com/eclipse-dirigible/dirigible/issues/8623
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8845

## The problem

Some status moves must be refused while a row of another entity exists:

1. A fiscal month must not be closed while a draft journal entry is dated inside it. Once the month
   is closed, the entry can no longer be posted. The entries belong to the month by their date, not
   through a reference.
2. An opportunity must not be marked lost while one of its quotations is accepted or ordered. The
   quotation references the opportunity, and the quotation's model uses the opportunity's model, so
   the opportunity's model cannot use the quotation's model back.

Nothing in the format says this. A transition's guard reads only the record's own fields.
`forbidWhen` and `requiredWhen` read the record and one hop. `kind: exists` (proposal 0135) is the
positive form: it refuses a write while NO matching row exists. Both moves are accepted.

## The proposed shape

```yaml
- name: FiscalPeriod
  checks:
    - kind: notExists
      entity: JournalEntry
      where: ["Status == DRAFT", "date >= fromDate", "date <= toDate"]
      status: CLOSED
      message: "{count} draft entries are dated in this period - post or delete them before closing it"

- name: Opportunity
  checks:
    - { kind: notExists, entity: Quotation, model: quotations, match: { Opportunity: id },
        where: "Status in [2, 3]", status: LOST,
        message: "A deal with an accepted or ordered quotation is not marked lost" }
```

## Expected behaviour

- The move into the `status` gate is refused, with `message`, while at least one row of `entity`
  satisfies every `match` pair and every `where` term. `{count}` in the message is the number of
  such rows.
- A transition, a process step and a user's edit that write the gate status are all refused the
  same way.
- A row whose status or date does not satisfy `where` does not block the move. Once the blocking
  rows are gone or no longer match, the move is accepted.

## Edge rules

- `status` is required: the check is about a move.
- At least one of `match` and `where` is required.
- `match` maps a property of `entity` to a path on the record, as on `exists`. A to-one of `entity`
  that points at the record's own entity matches the record's primary key (`match: { Opportunity: id }`).
- A `where` term is `<property of entity> <op> <operand>`, with `op` one of `==`, `!=`, `>=`, `<=`,
  `>`, `<`. The operand is a literal (a number, a quoted string, a boolean) or a path on the record.
  `== null` and `!= null` test for an empty value. `<property> in [a, b]` matches any of the listed
  literals. A status name on the target's status relation names one of the target's own statuses.
- An ordering compares dates, timestamps or numbers. A date is ordered against a path on the
  record, not against a literal.
- A `match` value or `where` path that is empty on the record skips the check.
- `model` may name a model the file does not `uses:`. The target's rows are then found by that
  model's name and the entity's name when the check runs. A model that is not deployed has no rows,
  so the move is accepted. Its property names cannot be checked when the file is read, and its
  statuses are written as numeric ids.
- A matching row created while the move is being written either is counted, or is stored after the
  move.

## Prior art / workarounds

A hand-written action on the record that counts the other entity's rows and throws, or a
precondition on the button that the other entry points (a process step, a direct edit) do not
apply. Reference implementation: https://github.com/eclipse-dirigible/dirigible/pull/8845.

## Specification text

**Anchor:** Entities & fields > checks - declarative validations, a new `#### kind: notExists`
section after `#### kind: exists — a write that needs a matching row of another entity` (proposal
0135).

#### kind: notExists — a status move refused while a matching row of another entity exists

A `notExists` check names another entity and the rows of it that refuse a status move. The move
into the `status` gate is refused while one of those rows exists:

```yaml
- name: FiscalPeriod
  checks:
    - kind: notExists
      entity: JournalEntry
      where: ["Status == DRAFT", "date >= fromDate", "date <= toDate"]
      status: CLOSED
      message: "{count} draft entries are dated in this period"
```

- `entity` — the entity the refusing rows belong to; with `model`, an entity of the model of that
  name, which the file need not `uses:`.
- `match` — optional; each property of `entity` and the path on the record its value must equal, as
  on `exists`. A to-one of `entity` to the record's own entity matches the record's primary key.
- `where` — optional; a term or a list of terms over the rows of `entity`:
  `<property> <op> <literal or path on the record>` with `op` one of `== != >= <= > <`,
  `<property> == null`, `<property> != null`, or `<property> in [<literal>, ...]`.
- `when` — optional; the condition under which the rule applies, in the `forbidWhen` grammar.
- `status` — the status whose move is refused.
- `message` — the authored text the refused move is reported with; `{count}` is the number of rows
  that refuse it.

> **Normative.**
> A conforming generator MUST refuse a write that persists the record carrying `status`, with
> `message`, when `when` holds (or is absent), every `match` value and `where` path is present, and
> at least one row of `entity` satisfies every `match` pair and every `where` term. It MUST apply
> this to every writer of the status: a transition, a process step and a user's edit. It MUST pass
> the values as bound parameters, never as query text. A row of `entity` created while the move is
> written MUST either be counted by the check or be stored after the move commits. A status name in
> a `where` term on the target's status relation MUST resolve against the target's statuses. With a
> `model` the file does not `uses:`, the rows MUST be found by that model's and the entity's names
> when the check runs, and a model that is not deployed MUST be treated as holding no rows. Each of
> the following MUST be reported as an authoring error: a missing `entity`, `status` or `message`;
> neither `match` nor `where`; an `entity` of the file that is not declared, or is the record's own
> entity; a `match` or `where` path that does not resolve; a property the file's own target does not
> declare; an ordering of a value that is not a date, a timestamp or a number; a literal that is not
> a value of the property's type; `verbs`; and `severity: warn`.

## DSL index

| Construct | What it does |
| --- | --- |
| [`checks: kind: notExists`](#checks--declarative-validations) | refuses the move into `status` while a row of another `entity` matches `match` and `where`; `{count}` in `message` is the number of such rows |
