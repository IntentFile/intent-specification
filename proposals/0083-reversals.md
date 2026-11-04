# A manual Reverse creates a posted document's negated twin, linked both ways, in one unit of work

- **Status:** released in [1.12](../versions/1.12.md)
- **Issue:** https://github.com/IntentFile/intent-specification/issues/144, https://github.com/eclipse-dirigible/dirigible/issues/7886
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7900

## The problem

1.11 reverses only what a posting created. `postings` declares `reverses` for "a reversal / credit
when the source is voided": a second posting negates the items of the sibling it names when the
**source** reaches its void status. A posted, hand-written journal entry has no source and no source
event, so nothing in the file can reverse it.

A `generates` create-from from the entry to itself comes close - a button, a status guard, one unit
of work - but it can only copy. It cannot negate a value, and it cannot write anything onto the
record it was started from. So the accountant who has to storno a posted entry retypes it by hand,
with every amount negated, and links the two by hand, or does not. 1.11 has no `reversals` key.

## The proposed shape

```yaml
reversals:
  - name: reverseEntry
    forEntity: JournalEntry
    when: "Status == POSTED"            # the statuses the original may stand in
    link: Storno                        # on the twin: the original
    backLink: ReversedBy                # on the original: the twin
    negate: [debit, credit, currencyAmount]
    defaults: { date: now }
    refuse:
      - { when: "ReversedBy != null", message: "This entry is already reversed" }
      - { when: "Storno != null",     message: "A reversal cannot be reversed - reverse the original entry" }
```

## Expected behaviour

- A reversal **is a create-from** from and to the same entity. The button on the record, the status
  gate (`when` is the create-from's `fromStatus` guard), the single unit of work, numbering, the
  initial status, the process trigger and the period lock are the create-from's own.
- Every copyable header and line property is copied onto itself. `link` receives the original's key
  on the twin. Each decimal named in `negate` is copied negated; a line keeps its side, so a debit
  line of the original is a debit line of the twin with the negated amount. `defaults` apply as on a
  create-from.
- After the lines, `backLink` is written onto the original, as a system write: `immutableWhen`
  guards user writes only and does not refuse it, so a posted, frozen original still receives it.
- Each `refuse` entry is decided over the loaded original **under a lock on it**, so two concurrent
  clicks cannot both pass. A refusal is a conflict (409) carrying the authored message, and nothing
  is created.
- A record outside the `when` statuses is refused by the status gate (409), as a create-from's
  `fromStatus` refuses it.
- A twin dated into a closed period is refused by the period lock: no twin is created and no
  back-link is written.
- Once the twin is posted, the original and the twin net to zero.

## Edge rules

- Not copied: the primary key, `unique` fields, a `number:` field, calculated and aggregate fields,
  the status relation, a personal owner, `link` and `backLink` themselves, a line's composition
  relation to its master, and every key named in `defaults`.
- `refuse[].when` uses the record-local `forbidWhen` vocabulary, read over the original.
- Status names in `when` and in `refuse` resolve by seeded name.
- `negate`, `link`, `backLink` and `refuse` exist on a reversal only; spelled on a `generates` entry
  they are unknown keys and are reported.
- The button is hidden by the status gate only. On an original that is already reversed it is still
  offered, and the click answers 409 with the `refuse` message.
- A model without `reversals` generates exactly as before.

## Prior art / workarounds

The storno retyped by hand, every amount negated, and the two entries linked by hand where the model
had a field for it. A create-from copied the entry but left the sign and the back-link to the user.
Reference implementation: https://github.com/eclipse-dirigible/dirigible/pull/7900.

## Specification text

**Anchor:** Declarative glue > postings - source document to ledger, a new section after its
subsection "An amended source rewrites its posting" and before "posts - derived rows on an event".

### reversals - a posted document's negated twin

`reverses` reverses what a posting created, when its source is voided. A document that was written
by hand and then posted - a manual journal entry - has no source to void. `reversals` declares the
correction the accountant performs on it directly: a Reverse button that creates the document's
negated twin and links the two both ways.

```yaml
reversals:
  - name: reverseEntry
    forEntity: JournalEntry
    when: "Status == POSTED"            # the statuses the original may stand in
    link: Storno                        # on the twin: the original
    backLink: ReversedBy                # on the original: the twin
    negate: [debit, credit, currencyAmount]
    defaults: { date: now }
    refuse:
      - { when: "ReversedBy != null", message: "This entry is already reversed" }
      - { when: "Storno != null",     message: "A reversal cannot be reversed - reverse the original entry" }
```

A reversal is a [create-from](#generates--create-from) from `forEntity` to itself, and everything a
create-from does it does the same way: it adds a button on the record, `when` is its
[`fromStatus`](#generates--create-from) guard, the twin is created in one unit of work through the
entity's own write path - so numbering, the initial status, a process trigger and the
[period lock](#immutableinperiod--period--period-locking) apply to it - and `defaults` set values as
on a create-from. What it adds is the reversal's own half:

- every copyable header and line property is copied onto itself, and each decimal named in `negate`
  is copied negated; a line keeps its side, so the twin of a debit line is a debit line;
- `link` receives the original's key on the twin, and `backLink` the twin's key on the original;
- each `refuse` entry is a condition over the original, in the record-local vocabulary of
  [`forbidWhen`](#kind-forbidwhen--a-write-refused-while-a-condition-holds), with the message that
  refuses the reversal while it holds - "already reversed" and "a reversal cannot be reversed" are
  the two every ledger needs.

Once the twin is posted, the original and the twin net to zero. The original itself is never edited
beyond its `backLink`; a correction to a closed period is the twin, dated in an open one.

> **Normative.** A conforming generator MUST treat a `reversals` entry as a create-from from and to
> `forEntity`, with `when` as its `fromStatus` guard, and MUST create the twin, its lines and the
> original's `backLink` in one unit of work. It MUST copy every copyable header and line property
> onto itself, MUST copy each property named in `negate` negated, keeping a line on its side, and
> MUST write the original's key into `link` on the twin. It MUST NOT copy the primary key, a
> `unique` field, a `number:` field, a calculated or aggregate field, the status relation, a
> personal owner, `link`, `backLink`, a line's composition relation to its master, or a property
> named in `defaults`. After the lines it MUST write the twin's key into `backLink` on the original
> as a write of the application, which `immutableWhen` does not refuse.
> Each `refuse` condition MUST be decided over the original read under a lock that a concurrent
> reversal of the same original waits for, so two concurrent invocations cannot both pass; a
> refusal MUST be answered as a conflict carrying the authored message, and MUST leave nothing
> created and the original unchanged. A twin the period lock refuses MUST likewise leave no twin
> and no back-link. Status names in `when` and in `refuse` MUST resolve as
> [status references](#status-references--name-not-number) do.
> `negate`, `link`, `backLink` and `refuse` MUST be accepted on a `reversals` entry only; on a
> `generates` entry each MUST be reported as an unknown key. A file declaring no `reversals`
> generates as it did before.

The Reverse button is offered on every record its `when` guard admits; a `refuse` condition does not
hide it, and an already reversed original answers the click with the condition's message.

## DSL index

| Construct | What it does |
| --- | --- |
| [`reversals`](#reversals--a-posted-documents-negated-twin) | a Reverse button on a posted record: a create-from to the same entity that negates the named amounts, links twin and original both ways, and refuses under a lock while a `refuse` condition holds |
