# A transition the record's owner runs from the personal surface - `personal: true`

- **Status:** released in [1.12](../versions/1.12.md)
- **Issue:** https://github.com/IntentFile/intent-specification/issues/159, https://github.com/eclipse-dirigible/dirigible/issues/8406
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8412

## The problem

An employee books a leave request on their own personal page and changes their mind. The request
declares a `Withdraw` transition, but 1.11 offers a transition on the regular surface only: its
endpoint is gated on the entity's write roles and loads any record by id, and no personal page
renders it. So the owner has no Withdraw on "My requests", and the only way to give them one -
granting a write role - would also let them withdraw a colleague's request.

The personal surface already promises that "reads are filtered to the caller's mapped record, the
owner FK is forced server-side on writes, and a foreign record is not found". A transition is a write
that surface does not yet carry.

## The proposed shape

```yaml
entities:
  - name: VacationRequest
    relations:
      - { name: Employee, kind: manyToOne, to: Employee, model: employees, required: true, personal: true }
      - { name: Status,   kind: manyToOne, to: VacationStatus, function: EntityStatus, init: 1 }
transitions:
  - name: WithdrawVacationRequest
    forEntity: VacationRequest
    from: [SUBMITTED, APPROVED]
    setStatus: WITHDRAWN
    label: Withdraw
    personal: true                       # the owner runs it from the personal surface too
```

## Expected behaviour

- A transition declaring `personal: true` gets a second run on the personal surface, beside the
  regular one.
- The personal run resolves the caller's identity record the way the personal surface does, and
  answers 403 when the caller has none.
- It answers 404 for a missing record and for a record whose personal relation is not the caller's,
  and it decides that before the `from:` and `when:` guards (409). A colleague's record is therefore
  indistinguishable from a missing one.
- The write is the regular run's write: the status flip and its `-transitioned` event, committed
  together.
- The answer is the personal surface's view of the record, so `sensitive` and `visibleTo` fields are
  withheld as on the rest of that surface.
- The personal list rows, the personal form and the personal document page offer the transition on
  a record whose status it accepts, and re-read the record after it ran.
- The regular endpoint and the regular pages are unchanged. A transition without `personal: true`
  generates exactly as before.

## Edge rules

- `personal: true` is refused unless the entity declares its own `personal: true` relation, and is
  refused when that relation is `personalReadOnly`.
- The regular run keeps its write-role gate; the personal run does not need a write role.
- The partner surface renders no record actions; `personal` concerns the personal surface only.

## Prior art / workarounds

A hand-written personal endpoint that re-implemented the ownership check, the guard and the status
write, plus a custom page or button on the personal surface; or a write role that let the owner
reach the regular endpoint - and every colleague's record with it. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8412.

## Specification text

**Anchor:** Declarative glue > transitions - guarded status flips, appended after the notify
paragraph.

#### personal - the owner runs the transition

A transition MAY declare `personal: true`. The record's owner then runs it from the
[personal surface](#personal-and-partner-surfaces) as well - the employee who withdraws their own
leave request - without a write role that would reach every other owner's records too:

```yaml
transitions:
  - { name: WithdrawVacationRequest, forEntity: VacationRequest, from: [SUBMITTED, APPROVED],
      setStatus: WITHDRAWN, label: Withdraw, personal: true }
```

The transition keeps its regular run, gated on the entity's write roles as before, and gains a
second one on the personal surface. The personal run resolves the caller's identity record as the
rest of that surface does and acts only on a record the caller owns: a record of another owner is
not found, exactly as a missing one, and that is decided before the `from` statuses and the `when`
guard are. The write itself is the regular run's - the flip and its `-transitioned` event - and the
answer is the record as the personal surface shows it. The personal pages offer the button on a
record whose status the transition accepts, and read the record again after it ran.

> **Normative.**
> `personal: true` MUST be refused on a transition whose entity declares no `personal: true`
> relation of its own, and on one whose personal relation is `personalReadOnly`. A conforming
> generator MUST give such a transition a run on the personal surface that answers 403 when the
> caller has no identity record, and 404 for a missing record and for a record whose personal
> relation is not the caller's - before the `from` and `when` guards, which answer 409 - so that a
> record the caller does not own cannot be told from one that does not exist. The personal run MUST
> make the same write as the regular run, including its `-transitioned` event, and MUST answer with
> the personal surface's view of the record, withholding `sensitive` and `visibleTo` fields as that
> surface does. The regular run, its role gate and the regular pages MUST be unchanged, and a
> transition without `personal: true` MUST generate exactly as before. `personal` concerns the
> personal surface only; the partner surface carries no record actions.

**Anchor:** Scoped surfaces & roles > Personal and partner surfaces, appended to the
`personal: true` bullet:

A [transition](#transitions--guarded-status-flips) declaring `personal: true` is run from this
surface by the record's owner, under the same ownership rule.

## DSL index

| Construct | What it does |
| --- | --- |
| [`transitions[].personal`](#personal---the-owner-runs-the-transition) | the record's owner runs the transition from the personal surface, on their own records only, with no write role |
