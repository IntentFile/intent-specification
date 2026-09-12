# A composition child may be read-only on a scoped surface independently of its parent

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7340
- **Discussion:** (this PR)

## Why

`personalReadOnly` is declared alongside `personal: true`, and a composition child may not carry
`personal:` at all — it inherits the owner's scope through its parent. So a child's scoped surface is
writable exactly when its parent's is, and nothing can say otherwise.

That is the wrong coupling whenever a person authors a **header** whose **lines a generator writes**.
A leave request is filed by the employee, so its scoped surface must be writable; its day rows are
written only by the approval flow, and each one charges an entitlement. The generated scoped
controller for the child therefore accepts a create from the owner's own draft, against any
entitlement in the tenant — a colleague's account included. The scoped page shows this as an **Add**
button on the items panel of a document whose items the person is not meant to author.

The shape recurs: an expense claim whose reimbursement lines a rule computes, an order whose
allocation rows a settlement writes.

Nothing else expresses it. `personalReadOnly` on the parent closes the header the person must author.
`sensitive: true` hides values on read and still accepts the write. An immutability rule keyed on the
parent's final statuses does propagate to the child, but a DRAFT parent is mutable by definition —
which is exactly the window this lives in.

## What this adds

`personalReadOnly: true` on the child's **composition** relation — the same key one level up, on the
edge the scope actually travels:

```yaml
- name: VacationDay
  relations:
    - { name: Request, kind: manyToOne, to: VacationRequest, composition: true, required: true,
        personalReadOnly: true }
```

Read as: *the scope still comes from the parent; the writes do not.* The effect is today's
`personalReadOnly`, applied to this child only — its scoped create / update / delete are refused and
its scoped surfaces render no write affordance, including the parent page's items panel and child
panels — while the parent's own scoped surface stays writable. The unscoped surface is untouched, and
so is every generator-written path, which does not go through the scoped controller.

## The normative half

The key is refused wherever it would be carried nowhere, because a dropped access declaration reads
as a grant: on a relation that is neither `personal: true` nor a composition, on a second composition
of the same entity (only the first is the ownership edge the scope travels), and on a child whose
master has no scoped surface to inherit.

`partner:` has no counterpart key today and gains none here.

## Prior art

Proven out by the reference implementation in eclipse-dirigible/dirigible#7354: the parse-time
refusals above, the child's scoped controller answering 403, and an end-to-end assertion that the
parent keeps Save/Delete on its header while the items panel and the child panels offer no Add.

## Specification text

The prose below is what a release folds into the next version document, at the anchor given. It was
written against `versions/1.2.md` and is carried here unchanged.

**Anchor:** Personal and partner surfaces — replacing the `personalReadOnly` bullet

- **`personalReadOnly: true`** makes a personal surface see-only: create / update / delete are refused
  and the scoped pages render without new / edit / delete. Declared **with `personal: true`** it
  closes the declaring entity's own surface — use it for records an owner may see but never author, a
  balance or a payslip. Declared on a **composition relation** it closes only that child's inherited
  surface, while the parent it inherits the scope from stays writable: the scope still comes from the
  parent, the writes do not. That is the shape of a header the owner authors whose lines a generator
  writes — a leave request whose day rows an approval flow charges against an entitlement — where
  closing the parent would close the header the person must author. The child's scoped pages, the
  parent page's items panel and its child panels all render without a write affordance.
  It is an authoring error to declare it on a relation that is neither `personal: true` nor a
  composition, on a second composition of the same entity, or on a child whose master has no personal
  surface to inherit.
