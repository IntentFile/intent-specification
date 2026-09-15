# A register lookup narrows its register, reads the document header and copies the found row's scalars

- **Status:** released in [1.9](../versions/1.9.md)
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7025 (and https://github.com/eclipse-dirigible/dirigible/issues/6812 for `where:`)
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/6858 (`where:`), https://github.com/eclipse-dirigible/dirigible/pull/7032 (paths and `copy:`)

## The problem

[`resolves:`](../versions/1.6.md#resolves--fill-a-relation-from-a-register-valid-on-a-date) (proposal
[0006](0006-register-lookup-resolves.md), released in 1.3) fills a to-one from the register row
whose validity period covers a date the record carries. Its own motivating list named "a price from the
price list valid on the order date" - and the construct as released cannot express it. Three gaps,
found the moment a real register was put behind it.

**A register keeps its history, and the lookup cannot see past it.** Every `match:` pair binds a
register column to a column of the record, so "and only the rows that are still valid" has no form at
all. A register is exactly the kind of table that keeps its corrections:

| id | vehicle | driver | validFrom | validTo | status |
|---|---|---|---|---|---|
| 1 | CA1234AB | Petrov | 2026-01-01 | 2026-06-30 | CANCELLED |
| 2 | CA1234AB | Ivanov | 2026-01-01 | 2026-06-30 | ACTIVE |

A fine on 2026-03-14 should resolve to Ivanov. Both rows match the vehicle and both cover the date, so
the lookup reports `ambiguous` and routes the fine to a person - for a register with exactly one right
answer. Every correction ever made earns that, quietly, and it gets worse the longer the register lives.

**The keys the lookup needs are on the document, not on the line.** A sales-invoice line's price list
is its header's customer's (`SalesInvoiceItem -> SalesInvoice -> Customer -> PriceList`) and the date
in force is the header's issue date. Neither is a column of the line, and the 1.6 text requires the
right side of every `match` pair and `between.value` to be a property of the record. The only
workaround, copying the header's list and date down onto every line with `dependsOn`, is a UI-time
copy: a create over the API, a `generates:` create-from (proforma to invoice) and a schedule fan-out
(a recurring template) never run it, so the lines produced by exactly the automated paths stay
unpriced - while the interactive path looks correct, so nothing reports it.

**The value the business needs is a scalar of the found row, and nothing writes it.** The lookup
writes the relation, the outcome and the status. The price is `PriceListItem.price`, and it belongs on
the line's own `price` field - again reachable only through `dependsOn`, again UI-only.

The three gaps have one shape in common: the lookup knew only its own record and wrote only a link.

## The proposed shape

Three optional keys on a `resolves:` entry, and a path where a bare property was required:

```yaml
resolves:
  - name: priceFromList
    event: { onCreate: SalesInvoiceItem }
    set: priceListItem                                # the line points at the price-list ROW
    from: PriceListItem                               # PriceList x Product x validity x price
    match:
      product: product                                # the line's own column
      priceList: salesInvoice.customer.priceList      # a to-one PATH off the line: the header's customer's list
    between: { start: validFrom, end: validTo, value: salesInvoice.issuedOn }   # the header's date
    where: { status: ACTIVE }                         # constant register filter, ANDed onto the match
    copy: { price: price }                            # register field -> record field, on found only
    outcome: pricing
    notFound:  { setStatus: UNPRICED }
    ambiguous: { setStatus: UNPRICED }
```

And the driver register from above, narrowed to its valid rows:

```yaml
resolves:
  - name: identifyDriver
    event: { onCreate: Fine }
    set: driver
    from: VehicleAssignment
    match: { vehicle: vehicle }
    where: { status: ACTIVE }
    between: { start: validFrom, end: validTo, value: violationAt }
```

- `where: { <register property>: <literal>, ... }` - one or more constant conditions on the register,
  ANDed onto the match and the period.
- A `match:` value and `between.value` may be a **to-one path off the record**
  (`salesInvoice.customer.priceList`, `salesInvoice.issuedOn`) instead of a property of it.
- `copy: { <register field>: <record field>, ... }` - scalars of the covering row written onto the
  record when exactly one row covers.
- `set:` may point at the **register itself**, for a value-bearing register: the row carries the price,
  so the row is what the line links to.

## Expected behaviour

### `where:` narrows the register

Every `where` pair is one more condition every candidate row must satisfy, alongside the match and
the period. Several pairs are ANDed. A pair naming the register's `function: EntityStatus` relation
may give the status by its **seeded name**, and that name is resolved on the **register's** status
nomenclature, not the record's - the two entities have different lifecycles, and the record's would
hand back a plausible id from the wrong one. Any other pair is a literal compared as written; a
to-one condition compares the foreign key.

The filter is part of what the lookup checked, so a `notFound` produced by a filter that is too narrow
reads as a filter, not as missing data: the filter is named in the lookup's trace of what it checked.

### An operand may be a path off the record

A `match:` value or `between.value` that contains a `.` is a path walked from the record through its
to-one relations. Every segment but the last names a to-one relation of the entity reached so far;
the last names a field - or a to-one relation, whose foreign key is then the value compared. A path is
resolved on **every** write path the event fires on - API, UI, create-from, schedule, arrival - which
is the whole reason it is a path and not a column copied onto the line.

A path that meets a missing link (a line with no header yet, a customer with no price list) yields
**no value**, and the lookup reports `notFound` for that record. A broken link is a business fact
about the record - there is no list to price it from - and the record joins the worklist like any other
unresolved one, rather than failing the write that raised the event.

Two operands through the same prefix read the related record once: a line whose list and date both
come through `salesInvoice` loads the header a single time. A cross-model relation may only be the
**last** hop of a path - the record stores a projection of a foreign-model entity's own properties,
not its relations, so there is nothing left to walk on past it.

A bare property (no `.`) is not a path and keeps the meaning it had: a model written against 1.6
neither becomes invalid nor changes what it generates.

### `copy:` writes what the found row names

`copy` is for the values the found row **names**, as opposed to the relation it points at (which
`set:` fills). On `found` - and only then - each `<register field>: <record field>` pair writes the
covering row's field onto the record's field. The write is **per field** and never overwrites: a
record field that already carries a value is skipped while the rest of the copy applies, so a price a
person typed survives the lookup that would have priced the line.

The copied scalars are written together with the resolved relation and the outcome, in the same
targeted write. A record never ends up pointing at a price-list row with no price on it: either the
result landed whole or it did not land.

### `set:` may point at the register itself

Under 1.6 the register must carry exactly one to-one relation to the entity `set` points at, and that
relation is the value copied - the driver the assignment names. A value-bearing register has no such
relation: the price-list item **is** the value, and what the line needs is a link to the row it was
priced from. So `set:` may name a to-one whose target is the register itself, and the resolved value is
then the covering row's own key. The exactly-one-to-one rule applies only when `set` points elsewhere.

### The three outcomes, with a copy declared

| Outcome | Relation | Copied scalars | `outcome` field | Status |
|---|---|---|---|---|
| exactly one covering row | set to the row (or the row's own to-one) | written, per field, into empty fields only | `found` | `found.setStatus`, if declared |
| no covering row | left unset | nothing written | `notFound` | `notFound.setStatus` |
| more than one covering row | left unset | nothing written | `ambiguous` | `ambiguous.setStatus` |

A `notFound` covers three causes the trace tells apart: no row matched the keys and filter, no
matching row covered the date, or a path operand had no value. `ambiguous` is never resolved by a copy:
a generator that cannot choose the row cannot choose the price either.

### Re-delivery

A record that already carries the relation is skipped - the whole lookup, including its copy. A
re-delivered event, a second `onUpdate` on a line already priced, and a manual correction all meet the
same rule: nothing is rewritten. The per-field skip of `copy` is the same rule applied to each copied
field: on a record that does not yet carry the relation, a field a person has already filled keeps
its value while the empty ones are filled.

## Edge rules

**Where the 1.6 text and the shipped behaviour disagree.** Two sentences of the released section are
contradicted by the implementation and are replaced by the Specification text below:

1. *"`match` at least one pair whose left side is a property of the register and whose right side a
   property of the record"* and *"`between.value` MUST name a date field of the record"* - the right
   side may be a to-one path off the record; `between.value` may be a path ending at a date or
   timestamp field.
2. *"The register MUST carry exactly one to-one relation to the entity `set` points at; that relation
   is the value the lookup copies. Zero or more than one MUST be rejected"* - when `set` points at the
   register itself, no such relation is required and the resolved value is the row's own key.

A third sentence is extended rather than contradicted: *"The resolved relation, the outcome and the
status MUST be written as one targeted update"* now also covers the copied scalars, which ride the
result write. (Whether the routing status is written in that same update or in a separate one after
it is the subject of a separate change to the reference implementation, dirigible #7029, and not of
this proposal; the text below speaks only of the relation, the copies and the outcome.)

**Refused at parse - `where:`**

- a key that is not a field or to-one relation of the register;
- a value that is not a scalar literal (a null, a list or a map);
- a key that repeats a `match` key (compared case-insensitively). On a column already bound to the
  record a literal either says the same thing twice or contradicts it into matching nothing, and which
  one it is depends on data the parser cannot see. Refused rather than ANDed.

**Refused at parse - a path**

- a blank operand or an empty segment (`a..b`, a trailing `.`);
- a middle segment that is not a to-one relation of the entity reached;
- a segment after a cross-model relation - a cross-model relation can only be the last hop;
- a terminal segment the walk's target does not declare as a field or to-one relation;
- a `between.value` path whose terminal is not a `date` or `timestamp` field. A terminal that lives in
  another model carries no declared type at parse, so that one check falls to generation, exactly as
  a cross-model status nomenclature does.

Each refusal names the lookup, the operand and the segment that failed.

**Refused at parse - `copy:`**

- a source that is not a plain field of the register - a relation is refused with a message saying
  that a copy takes a scalar of the covering row and the relation the row points at is what `set:`
  fills;
- a target that is not a field of the record;
- a target that is the relation `set` fills, the `outcome` trace field, or the record's primary key;
- two register fields copied onto one record field - only one of them could ever win;
- a source and target of different declared types - a copy writes the value through unchanged, and a
  mismatch would otherwise surface only when the write reaches the store, inside a handler nobody is
  watching.

**Register relation rule, revised.** When `set`'s target is the register, the lookup is valid with no
to-one on the register at all. When it is any other entity, the register must carry exactly one
to-one relation to it - zero and two-or-more stay refused with a message naming the count.

**Interactions**

- A copied value is a targeted write and moves a document's totals like any other targeted write, but
  it does not publish the record's `-updated` event: an automatic write is not a person's edit. A
  consumer that must observe a copied value binds a `phases:` moment of the record instead.
- The relation-level `where:` on a user-picked to-one is a different construct: it is capped at one
  pair and filters a dropdown's options. The lookup `where:` takes several pairs and filters what the
  lookup may find; the two share only a name.
- `where:` does not change what `match:` may bind; the case that needs a register column compared with
  a record value stays a `match` pair, and the case that needs it compared with a constant is a `where`
  pair. The parser refuses the same column in both.

## Prior art / workarounds

The interim for pricing was the conditional `dependsOn` (`valueFrom: { by, cases, default }`): a
classifier on the header's customer picks **which column** of the product is copied (`wholesalePrice`
vs `price`). That yields one price per column, not one per product per list, and no validity period -
it is not a price list. And it is UI-only, so every automated create-from and schedule fan-out still
produced unpriced lines.

The interim for register history was to delete corrected rows, which is the history the register
existed to keep; or to leave the ambiguity and let a person resolve every fine on a vehicle that had
ever been re-assigned.

The reference implementation ships all three extensions (eclipse-dirigible/dirigible PR #6858 for the
filter, PR #7032 for paths and the copy, the latter closing issue #7025). Its authoring guide
documents the shape; this proposal carries it into the format so a second implementation can conform
to it rather than to that guide.

## Specification text

**Anchor:** Declarative glue > resolves — fill a relation from a register valid on a date

The text below replaces the section's example block and its first two `**Normative.**` paragraphs
(the event/`set`/`from`/`match`/`between` paragraph and the register-relation paragraph), keeps the
outcomes and observability paragraphs unchanged, replaces the final paragraph (skipping and the
targeted write), and appends the paragraphs on the filter, the path and the copy.

Set a to-one from the row of a **register** whose validity period covers a date the record carries. The register says "X applied to Y from A to B" — a vehicle assignment, a price list, a contract in force, an org assignment — and the record carries, or can reach through its to-one relations, the match key(s) and the date:

```yaml
resolves:
  - name: identifyDriver
    event: { onCreate: Fine }               # onCreate or onUpdate, optional `when` guard
    set: driver                             # the to-one of Fine this fills
    from: VehicleAssignment                 # the register
    match: { vehicle: vehicle }             # register property <- record property or to-one path (one or more)
    where: { status: ACTIVE }               # optional: constant register filter (one or more, ANDed)
    between: { start: validFrom, end: validTo, value: violationAt }
    copy: { rate: rate }                    # optional: register field -> record field, on found only
    outcome: resolution                     # optional string field: found / notFound / ambiguous
    found:     { setStatus: IDENTIFIED }
    notFound:  { setStatus: NO_MATCH }
    ambiguous: { setStatus: MULTIPLE_MATCHES }
```

> **Normative.** A lookup MUST declare exactly one of `onCreate` / `onUpdate` naming a declared entity; `onDelete` MUST be rejected, since there is no record left to fill. `set` MUST name a to-one relation of that entity, `from` a declared register entity, and `match` at least one pair whose left side is a property of the register and whose right side is a property of the record **or a to-one path off it** (below). `between.value` MUST name a date field of the record or a path that ends at one; `between.start` and `between.end` name date fields of the register and MAY each be omitted, in which case that side of the period is open. The end of a period is **inclusive**, and a bound expressed as a date (rather than an instant) covers its whole day.

> **Normative.** When `set` points at an entity other than the register, the register MUST carry exactly one to-one relation to that entity; that relation is the value the lookup resolves. Zero or more than one MUST be rejected — a register offering a choice of columns to resolve is a modelling ambiguity, and guessing one would defeat the construct's purpose. When `set` points at the **register itself** — the shape of a value-bearing register, whose row carries the value and is therefore what the record links to — no such relation is required, and the resolved value is the covering row's own key.

*(The two paragraphs on the three outcomes and on observability are unchanged.)*

A register keeps its history — a cancelled assignment stays beside the active one and keeps covering the same period — and a lookup that cannot tell them apart finds two rows and gives up as `ambiguous` for a register with exactly one right answer. `where:` narrows the register by constants the record does not carry:

```yaml
    where: { status: ACTIVE, kind: PRIMARY }
```

> **Normative.** Each `where` pair MUST name a property of the register and carry a scalar literal; a null, a list or a map MUST be rejected. Every pair is one more condition a candidate row MUST satisfy, ANDed with the match and the period; any number of pairs MAY be given. A pair naming the register's `function: EntityStatus` relation MAY give a [status name](#status-references--name-not-number), which MUST be resolved on the **register's** status nomenclature, never the record's. A `where` key that repeats a `match` key MUST be rejected: on a column already bound to the record a literal either repeats the match or contradicts it into matching nothing, and a conforming generator MUST NOT guess which. A conforming generator SHOULD name the filter in its record of what a `notFound` checked, so a filter that is too narrow reads as a filter and not as missing data.

The keys a lookup needs are often on the **document**, not on the record the event is about: a line's price list is its header's customer's, and the date in force is the header's. A `match` value or `between.value` MAY therefore be a **path** off the record through its to-one relations — `salesInvoice.customer.priceList`, `salesInvoice.issuedOn`:

```yaml
resolves:
  - name: priceFromList
    event: { onCreate: SalesInvoiceItem }
    set: priceListItem                                # the line points at the price-list ROW
    from: PriceListItem                               # PriceList x Product x validity x price
    match:
      product: product                                # the line's own column
      priceList: salesInvoice.customer.priceList      # a path off the line
    between: { start: validFrom, end: validTo, value: salesInvoice.issuedOn }
    where: { status: ACTIVE }
    copy: { price: price }                            # the scalar the found row NAMES
```

> **Normative.** An operand containing a `.` is a path. Every segment but the last MUST name a to-one relation of the entity reached so far; the last MUST name a field of it, or a to-one relation, whose foreign key is then the value compared. A blank operand, an empty segment, a middle segment that is not a to-one, a terminal the walked-to entity does not declare, and a `between.value` path whose terminal is not a `date` or `timestamp` field MUST each be rejected, naming the lookup, the operand and the failing segment. A cross-model relation MAY appear only as the **last** hop, since the record holds a projection of the foreign entity's own properties and not its relations; a segment after one MUST be rejected. A path MUST be resolved on every write path the event fires on, never only on an interactive one. A path that meets a missing link yields no value, and the lookup MUST then report `notFound` for that record rather than fail the write that raised the event. Two operands sharing a prefix SHOULD read the related record once. A bare property (no `.`) is not a path and MUST keep the meaning it had before this version.

`copy:` is for the values the found row **names**, as opposed to the relation it points at: `{ <register field>: <record field> }`, one or more pairs.

> **Normative.** A copy is written on `found` only — when exactly one row covers — field by field, and MUST NOT overwrite: a record field that already carries a value MUST be skipped while the remaining pairs apply, so a value a person typed survives. On `notFound` and `ambiguous` nothing is copied; a generator that cannot choose the row MUST NOT choose the value either. Both sides of a pair MUST be plain fields of the same declared type; a relation on either side MUST be rejected — the relation the row points at is what `set:` fills. A target that is the relation `set` fills, the `outcome` field or the record's primary key MUST be rejected, and so MUST two register fields copied onto one record field.

> **Normative.** A record that already carries the relation MUST be skipped in its entirety — relation, copies and outcome — so a manual correction is never overwritten and a re-delivered event is a no-op. The resolved relation, the copied scalars and the outcome MUST be written as one targeted write of those columns only, leaving every other column of the record — and any concurrent write to it — untouched; a record MUST NOT be observable pointing at a covering row whose declared copies have not landed.

## DSL index

| Construct | What it does |
| --- | --- |
| [`resolves[].where`](#resolves--fill-a-relation-from-a-register-valid-on-a-date) | constant conditions ANDed onto a register lookup, so a register's history does not make it ambiguous |
| [`resolves[].copy`](#resolves--fill-a-relation-from-a-register-valid-on-a-date) | scalars of the covering row written onto empty record fields on `found` |
| [`resolves[].match` / `between.value` path](#resolves--fill-a-relation-from-a-register-valid-on-a-date) | a lookup operand read through the record's to-one relations — the document header from a line |
