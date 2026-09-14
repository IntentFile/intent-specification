# A write is rejected while a condition holds - `checks: forbidWhen`

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7275
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7326

## The problem

`checks:` can say that exactly one of several fields is filled (`exactlyOne`), that a document has
enough lines and that they balance (`itemsMin` / `itemsSumEqual`), that a total stays above a floor
(`guard`), and - in two open proposals - that two values of a row compare (`compare`, spec PR #68)
and that a value is required under a condition (`requiredWhen`, spec PR #70). Every one of them
*requires*, *compares* or *counts*. None of them says the simplest rule of all: **this write is
refused while `<condition>` holds.** And no check's condition can read a value one hop away
through a to-one relation - so a child cannot ask about its parent.

The two gaps together leave a very common rule with no form. A sales invoice owns its payment
allocations as a composition child. Once the invoice is PAID, no further allocation may be added
to it - the clerk who tries should be told "Cannot add a payment to a fully paid invoice", and
better still should not be offered the button. Nothing in the format says it:

- [`immutableWhen`](../versions/1.6.md#immutablewhen--immutable--user-write-immutability) freezes
  a record for *editing* and reads the record's **own** status. A new allocation is a create, not an
  edit, and an allocation has no status of its own - the status that matters belongs to the
  invoice.
- [`locksWithMaster`](../versions/1.6.md#lockswithmaster--a-child-collection-that-outlives-its-masters-lock)
  is all-or-nothing. The allocation collection declares `locksWithMaster: false` on purpose, so
  money keeps being recorded while the invoice is ISSUED, SENT or PARTIAL - exactly the statuses
  allocations happen in. Turning the lock back on would refuse every allocation, not only the ones
  against a paid invoice.
- A `rollups:` `capacity:` guard refuses an *over*-allocation: a PAID invoice has zero remaining
  capacity, so a positive amount is rejected. That is a money-safety side effect, not a rule that
  says "the invoice is paid"; it lets a zero-amount row through, and its message is about
  capacity, not about the document.

So the rule is either absent - the button is offered, the form is filled, and a capacity message
comes back that says the wrong thing - or hand-written below the model layer, where it is
invisible to the file that is supposed to be the source of truth.

## The proposed shape

A new `checks:` kind that carries a condition and a message and nothing else. Its one reach beyond
`requiredWhen`'s condition is that a term may name a one-hop `Relation.field`, so a child tests its
parent:

```yaml
- name: SalesInvoice
  relations:
    - { name: Status, kind: manyToOne, to: SalesInvoiceStatus, function: EntityStatus, init: DRAFT }

- name: SalesInvoiceCustomerPayment          # the payment allocation - a composition child
  locksWithMaster: false                     # money is recorded while the invoice is ISSUED / SENT / PARTIAL ...
  checks:
    # ... but not once it is PAID: the write is refused, and the panel stops offering "Add"
    - { kind: forbidWhen, when: "SalesInvoice.Status == PAID",
        message: "Cannot add a payment to a fully paid invoice" }
  relations:
    - { name: SalesInvoice, kind: manyToOne, to: SalesInvoice, composition: true, required: true }
    - { name: CustomerPayment, kind: manyToOne, to: CustomerPayment, model: customer-payments }
```

The condition may also be record-local, and - like every row check that has one - may carry a
`status:` gate that moves the rule from "every write" to "the write that carries that status":

```yaml
- name: VacationRequest
  checks:
    # a request flagged as cancelled by the employee cannot be submitted for approval
    - { kind: forbidWhen, when: ["withdrawn == true", "Status == SUBMITTED"],
        message: "A withdrawn request cannot be submitted" }
    # the same rule, evaluated only when the record is persisted carrying SUBMITTED
    - { kind: forbidWhen, when: "withdrawn == true", status: SUBMITTED,
        message: "A withdrawn request cannot be submitted" }
```

| Key | Required | Meaning |
| --- | --- | --- |
| `kind` | yes | `forbidWhen` |
| `when` | yes | one `<Property> ==\|!= <literal>` term, or a list of them, **all** of which must hold for the write to be refused. `<Property>` is a field or to-one relation of the record, **or** a one-hop `<Relation>.<field>` over a to-one relation of the record - the relation's target may belong to another model. `<literal>` is a number, a quoted string, a boolean, a bare word, or a seeded status name |
| `message` | yes | the reason the write is refused, shown to the person who attempted it |
| `status` | no | the gate: with it, the rule holds when the record is persisted **carrying** that status, not on every write. Needs the entity's `function: EntityStatus` relation |

A `forbidWhen` carries **no** `field`, `value`, `op`, `than` or `fields` - it names no value, only the
condition under which the write is refused.

## Expected behaviour

**The refusal.** While every term of `when` holds, a **create or update** of the row is refused and
the write leaves nothing behind. The refusal is a validation outcome carrying the authored
`message` - the same kind of answer `exactlyOne` or a violated `required` gives - never a fault of
the platform. It is enforced on every surface the generator produces for user writes, so it holds
for the master-detail panel, the record's own form, an import, and a client calling the API
directly.

**The routing.** Without a `status:` gate the rule holds on every user write. With one, the rule is
evaluated when the record is persisted carrying that status - at the transition, not on the draft
before it - and it is evaluated on the same path the transition runs on, so the refusal reaches
the person who pressed the button rather than a background log. This is the split every gated row
check follows; `forbidWhen` adds nothing to it.

**The one-hop term.** A term `<Relation>.<field>` reads the related row the record's foreign key
points at: the generator loads that row first, then compares its field. A relation that is not set
(no foreign key on the record yet) reads as *no value*: a `==` term against a real value then does
**not** hold, and a `!=` term does. The hop is exactly one; the relation must be a to-one; its target
may be an entity of another model.

**Status names.** A literal compared against a status relation may be the seeded name. In a
record-local term (`Status == SUBMITTED`) the name resolves against the record's own nomenclature,
as everywhere. In a one-hop term whose field is the **target's** status relation
(`SalesInvoice.Status == PAID`) the name resolves against the **relation target's** nomenclature -
`PAID` is a status of the invoice, not of the allocation. Where the target belongs to another model
its nomenclature is seeded there and a name cannot be resolved; the reference is an authoring
error directing the author to the numeric id, exactly as the existing status-reference rule says.

**The affordance.** When the entity is a composition child and **every** term of the condition reads
its composition master (`<Master>.<field>`, over the composition relation), the generator hides the
child's **Add**, **row edit** and **row delete** affordances on the master-detail panel while the
condition holds against the master record that panel already displays. No further read is needed:
the panel holds the master. The server refusal stays in force underneath - hiding is the courtesy,
refusing is the rule. A condition with any record-local term, or a term over a relation that is not
the composition master, is enforced server-side only: the panel does not hold that other record,
and guessing would be worse than a refusal at save.

## Edge rules

- **Refused at generation** - each because the alternative is a rule that looks authored and is
  silently off:
  - a `field` (or any other value-naming key) on a `forbidWhen` - it reads no value;
  - a missing or blank `message` - the message is the whole point;
  - a missing or empty `when`;
  - a term that is not `<Property> ==|!= <literal>` or `<Relation>.<field> ==|!= <literal>`;
  - a term whose path does not resolve: an unknown property or relation, a relation that is not a
    to-one, or a second hop;
  - a term comparing a property whose type an equality is not exact on - a condition compares a
    string, an integer, a boolean, or a to-one by its key; a decimal, a date or a text blob is
    refused;
  - a literal that is not a value of the compared type (`Status == "seven"`, `withdrawn == 3`);
  - a status **name** in a term whose relation target belongs to another model - the numeric id is
    required there;
  - a `status:` gate on an entity without a `function: EntityStatus` relation.
- **`when` is a conjunction.** A list of terms must all hold. There is no `||`; a rule that should
  fire under either of two conditions is two `forbidWhen` checks with the same message - any check
  that holds refuses the write.
- **Delete is not a write of the row's values and is not refused by this check.** A `forbidWhen`
  covers create and update. Whether a row may be deleted while its master is in a status is the
  territory of `immutableWhen` (on the row) and `locksWithMaster` (through the master). The
  master-detail panel nevertheless hides the row **delete** affordance together with Add and edit
  while the condition holds against the master - the panel's three affordances are one gesture set,
  and a deletable row inside a paid invoice whose additions are refused would read as a defect. A
  generator that hides delete here MUST NOT let that hiding stand in for a rule the file does not
  state: a delete arriving through another surface is not refused by `forbidWhen`.
- **Independent of the immutability constructs.** A `forbidWhen` on a child whose collection says
  `locksWithMaster: false` is the reason that opt-out is safe to make: the collection stays writable
  through the master's lock, and this check names the one status in which it is not. Where both an
  immutability rule and a `forbidWhen` refuse a write, either refusal stands; they are not ordered.
- **`requiredWhen`'s grammar is unchanged.** A dotted `when` term on a `requiredWhen` remains an
  authoring error; its condition stays record-local. Only `forbidWhen` reads through a relation in
  its condition.
- **Where a surface does not hold the master** - the child's own list page, if the file gives it one,
  or a client of the API - nothing is hidden and the server refusal is the answer.
- **The current version text.** 1.6's `checks` section says "`exactlyOne` runs on every user write;
  `itemsMin` / `itemsSumEqual` are gated on a status transition", as though those were the kinds
  and their routings. They are not exhaustive - `guard` already sits beside them, and `compare` and
  `requiredWhen` are proposed - so this proposal adds a subsection rather than rewriting that
  sentence; the release that folds the row checks in should generalise it to "a row-level check
  holds on every user write unless it carries a `status:` gate". 1.6's *Status references - name, not
  number* lists the places a seeded name may be written and names "a check's `status` / `setStatus`"
  but not a check's `when`; the Specification text below states that a `forbidWhen` condition
  accepts a name, resolved against the relation target's nomenclature for a one-hop term, and the
  release should add "a check's `when`" to that list. Neither is a contradiction; both are
  omissions this text closes for its own construct.

## Prior art / workarounds

Today the rule is approximated, not stated. The allocation module declares `locksWithMaster: false`
and relies on the `rollups:` capacity guard to refuse an over-allocation against a paid invoice -
which refuses the positive amount with a capacity message, lets a zero-amount allocation through,
and offers the "Add" button right up to the moment it refuses. The alternatives are all below the
model layer: a hand-written validator in the child's generated write path, or a listener on the
child's create event that deletes what should never have been written - each a place that knows
the status id and that a regeneration has to preserve.

The reference implementation ships the shape described here. A first, server-only attempt was
declined for that reason: without the affordance half the clerk still sees "+", fills the form and
is refused at save - the experience the format's status-guarded actions (the `fromStatus` proposal,
spec PR #64) set out to avoid. The construct as merged refuses on the server on every path and
hides the master-detail affordance where the panel already knows the answer.

## Specification text

**Anchor:** Entities & fields > checks — declarative validations, as a new `####` subsection
`kind: forbidWhen — a write refused while a condition holds`, placed after
`#### kind: guard — a precondition over an aggregate` and before
`### immutableWhen / immutable — user-write immutability`.

#### kind: forbidWhen — a write refused while a condition holds

A `forbidWhen` check refuses a create or update of the row while its condition holds, with an
authored message. It names no value - only the condition and the reason:

```yaml
- name: SalesInvoiceCustomerPayment          # a composition child of SalesInvoice
  locksWithMaster: false
  checks:
    - { kind: forbidWhen, when: "SalesInvoice.Status == PAID",
        message: "Cannot add a payment to a fully paid invoice" }
- name: VacationRequest
  checks:
    - { kind: forbidWhen, when: "withdrawn == true", status: SUBMITTED,
        message: "A withdrawn request cannot be submitted" }
```

`when` is one `<Property> ==|!= <literal>` term or a list of them, all of which must hold.
`<Property>` is a field or to-one relation of the record, or - the reach no other check's
condition has - a one-hop `<Relation>.<field>` over a to-one relation of the record, whose target
may belong to another model. A child may thereby refuse a write on the strength of its parent's
state. The literal is a number, a quoted string, a boolean, a bare word or a seeded status name;
in a one-hop term whose field is the target's status relation, the name is the **target's**
status. The optional `status:` gate routes the rule exactly as on every gated row check: without
it the rule holds on every user write; with it, when the record is persisted carrying that status.

Where the entity is a composition child and every term reads its composition master, the
generated master-detail panel hides the child's add, edit and delete affordances while the
condition holds against the master record it displays. The server refusal stands underneath on
every path.

> **Normative.**
> A conforming generator MUST refuse a create or update of the row while every term of `when`
> holds, with `message`, as a validation outcome and never as a fault, on every surface it
> generates for user writes. Without a `status:` gate the rule MUST hold on every user write; with
> one it MUST be evaluated when the record is persisted carrying that status, on the path the
> transition runs on, so the refusal reaches the person who acted. A one-hop term MUST read the
> related row the record's foreign key names; an unset relation reads as no value, so a `==` term
> against a value does not hold and a `!=` term does. A status name in a one-hop term MUST be
> resolved against the relation target's nomenclature; a name against a target owned by another
> model is an authoring error directing the author to the numeric id.
> When the entity is a composition child and every term reads its composition master, the
> generator MUST hide the child's add, edit and delete affordances on the master-detail panel while
> the condition holds against the displayed master, without an additional read, and MUST keep the
> server refusal in force regardless. A condition with a record-local term, or a term over a
> relation that is not the composition master, is enforced server-side only.
> A `forbidWhen` does not refuse a delete; delete under a master's status is `immutableWhen` /
> `locksWithMaster`'s rule.
> A generator MUST reject as an authoring error: a `field` or any value-naming key on a
> `forbidWhen`; a missing `message`; a missing or empty `when`; a term that is not
> `<Property> ==|!= <literal>` or `<Relation>.<field> ==|!= <literal>`; a path that does not
> resolve, walks more than one hop, or crosses a relation that is not a to-one; a compared
> property whose type is not a string, an integer, a boolean or a to-one key; a literal that is not
> a value of that type; a status name against a cross-model target; and a `status:` gate on an
> entity without a `function: EntityStatus` relation. `requiredWhen`'s condition remains
> record-local; a dotted term there stays an authoring error.

## DSL index

| Construct | What it does |
| --- | --- |
| [`checks: kind: forbidWhen`](#kind-forbidwhen--a-write-refused-while-a-condition-holds) | refuse a create or update while a condition over the row - or its parent, one hop away - holds, with a message; the master-detail affordance is hidden alongside |
