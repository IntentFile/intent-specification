# A duplicate refused at a status gate

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7750
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8841

## The problem

`duplicate` is a warning only. It asks the person saving when another row carries the same
`fields`, and the format rejects a `status:` gate on it. That is right while a record is drafted,
and it is not enough later on.

A purchase ledger enters supplier bills. Two DRAFT bills from one supplier with the same supplier
invoice number may coexist: a recurring schedule drafts bills with a placeholder number, and a clerk
re-keys a bill before deleting the first. But the second of them may not be APPROVED while the first
is APPROVED, PARTIAL or PAID: that is the same paper booked twice into the purchase ledger. A
`unique: [Supplier, supplierNumber]` key is wrong, because it refuses the drafts and keeps refusing
after a bill is cancelled. The gated checks (`requiredWhen`, `itemsMin`, `compare` with `status:`)
only read the record itself and its lines, never the other rows of the entity. So the rule has no
declaration.

## The proposed shape

A `duplicate` that carries a `status:` gate refuses the move into that status, counting only the
other rows that hold one of the statuses `among` lists:

```yaml
- name: Bill
  checks:
    # asked while drafting, as before
    - { kind: duplicate, fields: [Supplier, supplierNumber], message: "Already entered: {match}" }
    # refused at approval
    - { kind: duplicate, fields: [Supplier, supplierNumber], status: APPROVED,
        among: [APPROVED, PARTIAL, PAID],
        message: "This supplier invoice number is already approved for this supplier: {match}" }
```

## Expected behaviour

- When a record is stored carrying the gate status (a transition, a process step, an update), the
  write is refused with the authored message while another row of the same entity carries the same
  values in `fields` and holds one of the `among` statuses. Nothing is stored.
- The values are compared exactly as an ungated `duplicate` compares them: `string` / `text` members
  ignoring case and leading and trailing whitespace, every other member by value. The row being
  written never counts against itself. An absent value repeats nothing.
- `{match}` names the colliding record's display label.
- A row in a status `among` does not list (a cancelled bill, a draft) never refuses anything.
- Two writes that move two such rows into the gate status at the same time cannot both succeed.
- A record stored in any other status is not affected; the ungated warning, when declared, still
  asks at drafting time.

## Edge rules

- `among` is required when `status:` is present, and rejected without it. There is no default: any
  default would count a status the author did not name (a cancelled bill refusing the approval of
  its replacement), or miss one (a paid bill not refusing it), and the refusal is hard.
- `among` is rejected on every other check kind.
- `severity: warn` is rejected together with a `status:` gate on a `duplicate`, as on every gated
  check: nobody can confirm a warning at a transition. `severity: error` is accepted with the gate
  and is the default; on an ungated `duplicate` it is still rejected (that is `unique`).
- The gate and `among` name statuses of the record's own status nomenclature, by name or by id, as
  every other status reference does. The entity must declare a status relation.

## Prior art / workarounds

Before this, the rule was a hand-written check in the code of each application, run in the approval
step, usually without protection against two approvals at once. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8841.

## Specification text

**Anchor:** Checks > Warn and confirm (`severity: warn`, `duplicate`, `itemsCompare`), appended after
its normative block.

A `duplicate` with a `status:` gate is not a warning. It refuses the move into that status while
another row with the same values holds one of the statuses `among` lists:

```yaml
- { kind: duplicate, fields: [Supplier, supplierNumber], status: APPROVED, among: [APPROVED, PARTIAL, PAID],
    message: "This supplier invoice number is already approved for this supplier: {match}" }
```

> **Normative.**
> A `duplicate` that declares `status:` MUST be enforced as a refusal whenever the record is stored
> carrying that status, on every write path, and MUST refuse the write, storing nothing, when another
> row of the entity carries the same `fields` values - compared as an ungated `duplicate` compares
> them - and holds one of the `among` statuses. The row being written MUST NOT count against itself,
> an absent value MUST NOT match, and `{match}` MUST resolve to the colliding record's display label.
> Concurrent writes that would each pass the check alone MUST NOT both be stored. `among` MUST be
> required with `status:` and MUST be rejected without it and on every other kind; `severity: warn`
> MUST be rejected together with `status:` on a `duplicate`.

## DSL index

| Construct | What it does |
| --- | --- |
| [`checks: duplicate` + `status` + `among`](#warn-and-confirm--severity-warn-duplicate-itemscompare) | refuses the move into a status while another row with the same values holds one of the listed statuses |
