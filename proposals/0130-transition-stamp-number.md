# A transition issues the document number - `stamp: number`

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8493
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8526

## The problem

A document numbered `stampOn: issue` gets its number at the issue step of a process. A document
whose Issue is a transition button - DRAFT to ISSUED, with no process - cannot be numbered that
way, so it must number at create. A draft that is then deleted has already spent a value of the
series, and two deleted drafts between two issued documents leave a permanent gap (1, 2, 5) in a
series that must be gap-free.

## The proposed shape

```yaml
transitions:
  - { name: IssueProtocol,  forEntity: VatProtocol, from: [DRAFT], setStatus: ISSUED, stamp: number }
  - { name: CancelProtocol, forEntity: VatProtocol, from: [DRAFT], setStatus: CANCELLED }
```

## Expected behaviour

- A transition declaring `stamp: number` draws the entity's `stampOn: issue` number and writes it in
  the same write as the status.
- The move is checked before a number is drawn: its lifecycle edge and the checks gated on the
  target status. A refused move answers with the check's refusal and spends no number.
- A write refused after the draw returns the number. The series stays gap-free.
- A document already numbered keeps its number.
- Only the transition that declares the key numbers the document.
- The number travels in the transition's own `-transitioned` event.

## Edge rules

- `number` is the only value of `stamp`.
- `stamp: number` is refused on an entity that declares no `number: { stampOn: issue }` field.

## Prior art / workarounds

`stampOn: create`, which numbers every draft, including the ones later deleted. Reference
implementation: https://github.com/eclipse-dirigible/dirigible/pull/8526.

## Specification text

**Anchor:** Declarative glue > transitions - guarded status flips, a new subsection after
"personal - the owner runs the transition" (0098).

#### stamp - the transition issues the document number

A transition MAY declare `stamp: number`. It is then the document's Issue: the entity's
[`number: { stampOn: issue }`](#document-numbering) is drawn and written together with the
status, so a draft holds no number and a draft deleted before it is issued leaves no gap.

```yaml
transitions:
  - { name: IssueProtocol, forEntity: VatProtocol, from: [DRAFT], setStatus: ISSUED, stamp: number }
```

> **Normative.**
> `stamp` MUST be refused with any value other than `number`, and on a transition whose entity
> declares no `stampOn: issue` number field. A conforming generator MUST check the move - its
> lifecycle edge and the checks gated on the target status - before it draws a number, MUST write
> the number and the status in one write, and MUST return a drawn number when that write is refused,
> so that a refused move spends no value of the series. A document whose number field already holds
> a number MUST keep it. Only the transition that declares `stamp: number` draws the number.

**Anchor:** Entities & fields > Document numbering, the `stampOn` bullet. Its `issue` clause
becomes:

`issue` puts a placeholder on the field at create and stamps the real number when the process
reaches the wired step, or when a transition declaring
[`stamp: number`](#stamp---the-transition-issues-the-document-number) moves the record.

## DSL index

A new row, after `transitions[].personal`:

| Construct | What it does |
| --- | --- |
| [`transitions[].stamp`](#stamp---the-transition-issues-the-document-number) | the transition draws the entity's `stampOn: issue` number in the same write as its status, after its gated checks pass |
