# A reference may outlive its target - `whenTargetDeleted: keep` - and a reversal bound to the source's delete mirrors the original's stored lines

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/205, https://github.com/eclipse-dirigible/dirigible/issues/7634
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7673
- **Amends:** [0064 - when the target is deleted](0064-when-target-deleted.md), [0075 - an `onCreate` posting follows its source](0075-on-create-posting-follows-its-source.md)

## The problem

0075, released in 1.11, says that on the delete of an `onCreate` posting's source "it MUST reverse
the created document through `reverses` where one is declared". It does not say how, and 1.11 has
neither of the two keys that make it work.

- **The reference.** The created entry points back at its source through the posting's
  `backReference`, a to-one relation of the entry. `whenTargetDeleted` takes `restrict | nullify |
  cascade` (0064), and each of them breaks the reversal: `restrict`, the default, refuses the
  source's delete; `nullify` clears the link before the reversal can follow it; `cascade` deletes
  the original entry. Nothing lets the reference survive its target.
- **The trigger and the lines.** A `reverses` entry is bound to a transition, and it negates the
  sibling's items as derived from the source. A deleted source has no transition, and re-deriving
  from it is wrong anyway: a source edited after its entry was posted leaves a divergence that the
  posted entry still carries, and only negating what was actually posted nets the original to zero.

## The proposed shape

```yaml
entities:
  - name: JournalEntry
    relations:
      - { name: Payment, kind: manyToOne, to: Payment, model: payments,
          whenTargetDeleted: keep }             # the back-reference outlives the payment

postings:
  - name: paymentPosting
    event: { onCreate: Payment, model: payments }
    creates: JournalEntry
    backReference: Payment
    items:
      - { Account: rule(bankAccount),       debit: "Amount" }
      - { Account: rule(receivableAccount), credit: "Amount" }
  - name: paymentStorno
    event: { onDelete: Payment }                # the sibling's source
    reverses: paymentPosting
    storno: Storno
```

## Expected behaviour

- A to-one declaring `whenTargetDeleted: keep` is left pointing at a target that no longer exists.
  The target's delete is not refused, the referencing rows are otherwise unaffected, and a read of
  the relation resolves to nothing.
- A `reverses` entry bound to `event: { onDelete: <source> }` runs when that source is deleted. It
  reads the deleted row as it was stored, finds the original entry through the kept back-reference,
  and creates the storno by **mirroring the original's stored lines**: every amount cell negated,
  every rule and dimension cell copied. It does not re-derive the lines from the source.
- A source edited after its entry was posted is still netted to zero by its delete: the storno
  negates what was posted, not what the source said last.

## Edge rules

- `keep` is accepted on a relation to a target owned by another model only. A same-model relation is
  a reference the database itself refuses to leave dangling, so `keep` on one is an authoring error.
- The sibling's `backReference` of an `onDelete` reversal MUST declare `keep`; `restrict` (including
  the default), `nullify` and `cascade` there are authoring errors, each naming why it breaks the
  reversal.
- `onDelete` without `reverses`, and an `onDelete` naming an entity other than the sibling's source,
  are authoring errors.
- A `reverses` entry bound to a transition behaves as in 1.11.

## Prior art / workarounds

None within the file. Under the default `restrict` the source's delete was refused while its entry
pointed at it; under `nullify` or `cascade` the reversal had no original left to follow. Reference
implementation: https://github.com/eclipse-dirigible/dirigible/pull/7673.

## Specification text

Two pieces of text, each at its own anchor.

**Anchor:** Relations & multi-model > Relation attributes, the `whenTargetDeleted` bullet and its
**Normative** block (0064), amended.

The bullet becomes:

- **`whenTargetDeleted: restrict | nullify | cascade | keep`** says what deleting the referenced
  record does to the rows of this entity that point at it. `restrict` (the default, also when the
  key is absent) refuses the delete while any row references the target; `nullify` clears the
  reference; `cascade` deletes the referencing rows with it; `keep` leaves the reference in place,
  pointing at a target that no longer exists - the reference outlives its target, which is what a
  [reversal bound to the source's delete](#an-amended-source-rewrites-its-posting) follows. The rule
  is declared on the referencing relation and holds across models; `keep` is accepted on a relation
  to a target owned by another model only.

The **Normative** block gains, after "... both in the same unit of work as the target's delete, so a
failure leaves the target in place.":

> Under `keep` it MUST NOT refuse the target's delete and MUST leave the referencing rows, their key
> included, unchanged; a read of the relation then resolves to no target. A generator MUST reject
> `keep` on a relation whose target is owned by the declaring model.

**Anchor:** Declarative glue > postings - source document to ledger > An amended source rewrites its
posting, the paragraph and **Normative** block on `onCreate` (0075), amended by appending:

The reversal of a deleted source is a `reverses` entry bound to the source's delete:

```yaml
postings:
  - name: paymentStorno
    event: { onDelete: Payment }      # the sibling's source
    reverses: paymentPosting
    storno: Storno
```

It reads the deleted row as it was stored and finds the original through the sibling's
`backReference`, which therefore declares
[`whenTargetDeleted: keep`](#relation-attributes) - every other rule loses the original before the
reversal can reach it. Unlike a reversal bound to a transition, it does not re-derive its lines from
the source: it mirrors the lines the original actually stores, so a source edited after its entry
was posted is still netted to zero.

> **Normative.** A `reverses` entry MAY bind `event: { onDelete: <Entity> }`, where `<Entity>` is the
> source of the sibling it reverses; a conforming generator MUST run it on the delete of that source,
> reading the deleted row as stored. It MUST create the reversal by mirroring the original's stored
> lines - every amount cell negated, every rule and dimension cell copied - and MUST NOT re-derive
> them from the source. The sibling's `backReference` MUST declare `whenTargetDeleted: keep`;
> `restrict`, declared or by default, `nullify` and `cascade` on it MUST be reported as authoring
> errors. `onDelete` on an entry without `reverses`, and an `onDelete` naming an entity other than
> the sibling's source, MUST be reported as authoring errors. A `reverses` entry bound to a
> transition is unaffected.

## DSL index

The `whenTargetDeleted` row is extended, and a row is added:

| Construct | What it does |
| --- | --- |
| [`whenTargetDeleted`](#relation-attributes) | on a plain to-one: what deleting the referenced record does to the referencing rows - `restrict` (default), `nullify`, `cascade`, `keep`; same-model and cross-model |
| [`whenTargetDeleted: keep`](#relation-attributes) | a cross-model reference that outlives its target: the delete is not refused and the key stays, so a reversal bound to the source's delete can still find the entry it reverses |
| [`postings` `event.onDelete`](#an-amended-source-rewrites-its-posting) | a `reverses` entry run on the delete of the sibling's source, mirroring the original's stored lines negated |
