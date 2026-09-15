# A `when` guard may be a list of comparisons, their AND

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/6957
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7006 (the list form),
  https://github.com/eclipse-dirigible/dirigible/pull/7317 (issue
  [#7289](https://github.com/eclipse-dirigible/dirigible/issues/7289): the same guard on
  `notifications` / `integrations` / `outbound` resolves a status name and refuses a guard that does
  not parse)

## The problem

A status is often reached by more than one path. A fine is ingested, a `resolves:` lookup finds the
vehicle's assigned driver and routes the record to `DRIVER_IDENTIFIED`; when the lookup finds nobody,
an officer identifies the driver by hand and a task sets the same status. Downstream, one document must
exist once the driver is known, however the driver became known - the declaration - and that
create-from needs ONE converged status to bind to, because an at-most-once create-from is guarded by
the back-reference and two rules sharing a target and a back-reference are refused by design.

Now the audit requirement arrives: "append a success row to the log when the identification was
automatic." The only guard the event axis offered was a single comparison:

```yaml
event: { onTransition: Fine, when: "Status == DRIVER_IDENTIFIED" }
```

It fires on both paths. The lookup already stamps exactly the provenance needed - its `outcome:` trace
field carries `found` / `notFound` / `ambiguous` - but a guard could compare a status and nothing else,
so the trace could not be read. Splitting the status (`DRIVER_IDENTIFIED_AUTO` / `_MANUAL`) does not
compose: the declaration's converged guard breaks, and every later consumer must now name two statuses
for one fact. The 1.6 text is explicit about the wall: "two outcomes you must tell apart downstream
need two statuses", and the requirement "log which path identified the driver" was simply not
expressible.

A second, quieter failure lived on the same key. On the three reacting glue lists the guard was
neither resolved nor checked: `when: "Status == ISSUED"` on a `notifications[]` entry compared the
integer status key with the text `ISSUED` and never held - the mail never went out, with everything
green - while a guard that did not parse (`Status = ISSUED`, `status == 'ISSUED' and channel ==
'mail'`) was silently read as *true* and fired the reaction on every event. That is a guard nobody
authored, in both directions.

## The proposed shape

Wherever an event binding takes `when:`, the value may be a **list** of comparisons. Each element is
the one comparison the guard always took; the list means their AND:

```yaml
generates:
  # the document that must exist once, however the driver was identified: the bare converged guard
  - name: declaration-from-fine
    from: Fine
    to: Declaration
    event: { onTransition: Fine, when: "Status == DRIVER_IDENTIFIED" }
    map: { Fine: id, Vehicle: Vehicle }

  # the trail row for the AUTOMATIC path only: the status AND the lookup's trace field
  - name: log-driver-identified
    from: Fine
    to: FineLog
    event:
      onTransition: Fine
      mode: append
      when:
        - "Status == DRIVER_IDENTIFIED"
        - "resolution == found"
    map: { Fine: id }
    defaults: { kind: "DRIVER_IDENTIFICATION_SUCCESS", at: now }

notifications:
  - name: issued-by-mail
    event:
      onUpdate: SalesInvoice
      when:
        - "Status == ISSUED"          # the seeded name resolves here as at every other guard site
        - "sentMethod == 1"
    to: Customer.email
    subject: "Invoice {Number}"
    body: "Please find your invoice attached."
    attach: print
```

The single string stays valid everywhere and means what it meant. A list of one element is the same
guard as that element.

Nothing else is added. There is no `||`, no parenthesis, no operator beyond `==` and `!=`: the
restriction is encoded in the shape, not policed by an expression grammar, so there is no dialect to
grow one operator at a time.

## Expected behaviour

A conforming generator MUST evaluate every comparison of the list against the same record the single
guard is evaluated against, and run the reaction only when all of them hold. On a create-from that
record is the source as re-read at delivery (the existing rule); on the reacting glue lists it is the
event record the reaction is about - the entity a lifecycle binding names, or, for a process-step
binding, the record the process runs on.

Each element is resolved and typed on its own:

- an element comparing the record's status relation may name the status by its **seeded name** or its
  id; the name resolves against the nomenclature of the entity the event is about, before any other
  validation, exactly as a single guard's does;
- an element comparing another property is left as written: `resolution == found` compares the
  string field `resolution` with the text `found` - a bare word on the right of a string comparison is
  a string literal, quoted or not;
- every comparison is rendered against the property's **declared type**, so an integer key is never
  compared with a text and a comparison cannot be silently always-false.

The list form is available on the `event:` of a create-from (`generates`), on the `event:` of a
`notifications` / `integrations` / `outbound` entry, and on a process's `trigger:`. The construct the
list exists for - reading a lookup's `outcome:` trace beside the converged status - therefore reaches
every reaction that can observe a status write.

## Edge rules

- **An empty list is rejected.** It would guard nothing, which is a bare binding written the long way;
  the author is told to drop the key or fill the list.
- **An element that is not a comparison is rejected**, with the accepted shape in the message: a
  non-string element (a nested map, a number), and a string that does not parse as
  `<Property> ==|!= <literal>` - `Status = ISSUED`, a prose conjunction, a `||`. A guard that does not
  parse is a parse error, never *true*: the silent degradation is the failure the construct replaces.
- **A property the record does not carry is rejected.** The condition is read off the record itself;
  a one-hop path is not a property of it. On a create-from the message names the intended use - a
  string comparison guards one of the source's own fields, typically a lookup's `outcome:` trace field.
- **A literal that is not a value of the property's type is rejected**, and so is a property of a type
  no equality is exact on: only a string, a text, an integer, a long, a boolean and a to-one's key are
  guardable; a decimal or a date is compared for equality by nobody who means it. On a create-from the
  literal comparison is narrower still: it guards the source's **string** fields; the one numeric
  comparison the list carries is the status guard.
- **The same property twice is rejected.** A second `==` on one property can never hold together with
  the first; a second `!=` is redundant. Either way the author meant something else.
- **On an `onTransition` create-from the list MUST contain the status comparison** - the moment is a
  status reached, and a list of trace-field comparisons alone names no moment. That comparison is an
  equality, one per list; a second numeric comparison is rejected. On `onCreate` and on a process-step
  binding the whole guard stays optional, as before.
- **A status name in a cross-model nomenclature still cannot be resolved** (its seeds live in the
  other model); such an element is an authoring error directing the author to the numeric id, as every
  cross-model status reference is.
- **A guard on a field a user can edit is warned about, not refused.** A trace the platform writes
  (`readOnly: true`, as a lookup's `outcome:` target is) answers "how did this record get here"; a
  field a user can edit answers "what does it say today", and an edit then silently changes which
  automations fire. The generator reports it with the one-line fix (`readOnly: true`); it does not
  block the file.
- **Where a list is not accepted, it is refused with a message naming where it is** - not read as a
  single comparison, and not stringified. In the reference implementation a `postings[]` event guard
  and a `resolves[]` event guard take the single comparison only. The list form on `postings` is a
  natural next step and is NOT claimed by this proposal.
- **The three reacting glue lists now resolve a seeded status name and hold the guard to the grammar.**
  Before, `when: "Status == ISSUED"` on a `notifications[]` entry compared an integer key with a text
  (never true) and `Status = ISSUED` was read as *true*. Both are corrected by the same rule: a status
  name resolves against the entity the event is about, and a guard that does not parse is rejected. A
  1.6 file whose single guard was well-formed is unaffected; one that relied on the silent *true*
  becomes a parse error, which is the intended outcome.
- **Where the 1.6 text disagrees.** The `notifications` section says "`when:` supports a single
  `field ==|!= literal` guard" and the `trigger` section says the listener "applies the `when` guard
  (a single `field ==|!= literal`)". Both sentences are replaced below. The `resolves` section's
  reasoning that a downstream guard "compares a status and nothing else", so two outcomes must be
  told apart by two statuses, still holds for a `postings[]` guard and no longer holds for a
  create-from, a reacting glue entry or a trigger; the release should soften that paragraph to say so.
- **The reference implementation enforces the rejections above uniformly on a create-from.** On the
  three reacting glue lists it rejects the empty list, the unparseable element, the unknown property
  and the ill-typed literal, but does not yet refuse a duplicated property; on a process `trigger:` it
  accepts the list and renders every element it can parse, without the shape check. This proposal
  specifies the uniform rule; the two gaps are implementation follow-ups, not format choices.
- A `wait` step's `when:` and a `transitions[]` `when:` are unchanged by this proposal.

## Prior art / workarounds

Before the list form, the requirement was answered in one of three worse ways: a shared status plus a
log rule whose own description admits it fires on both paths; two statuses for one fact, which breaks
the converged at-most-once guard downstream and doubles every later reference; or a hand-written
listener that re-reads the trace field - code that knows the status ids and lives outside the model.
The alternative design considered and declined was an expression language for `when` (`&&`, `||`,
parentheses): it would grow one operator at a time, and the guard's earlier dialect drift across
sites came from exactly that.

## Specification text

**Anchor:** Declarative glue > The event axis — lifecycle events and process-step events (appended
after the Normative block).

Every binding of the axis MAY carry a **`when:` guard** inside its `event:` map, deciding per record
whether the reaction runs at all. It is one comparison `<Property> ==|!= <literal>` over the event
record's own properties - a field, or a to-one relation's key - **or a list of such comparisons,
meaning their AND**:

```yaml
notifications:
  - name: issued-by-mail
    event:
      onUpdate: SalesInvoice
      when:
        - "Status == ISSUED"        # the seeded name, resolved on SalesInvoice's status nomenclature
        - "sentMethod == 1"
```

The list is how converging paths to one status are told apart: a lookup routes a record to a status
automatically and stamps its `outcome:` trace field, a person's task sets the same status by hand, and
a reaction that must observe only the automatic path guards on the status **and** the trace. A list
cannot express an alternative, a negated conjunction or a grouping; what it cannot say is not part of
the format.

> **Normative.**
> A conforming generator MUST accept the guard as a single comparison string or as a list of them, and
> MUST run the reaction only when every comparison holds, all evaluated against the record the
> single guard is evaluated against. Each comparison is resolved on its own: one naming the record's
> status relation MAY name the status by its seeded name or its id, resolved on the nomenclature of
> the entity the event is about (the entity a lifecycle binding names; for a step binding, the
> trigger entity of the process) before any other validation; every other comparison is taken as
> written, a bare word on the right of a string comparison being a string literal. Every comparison
> MUST be rendered against the property's declared type. The generator MUST reject: an empty list; an
> element that is not a comparison string of the shape above; a property the record does not carry;
> a property whose type has no exact equality (only a string, a text, an integer, a long, a boolean
> and a to-one's key are guardable); a literal that is not a value of the property's type; and the
> same property compared twice. A guard that does not parse is an authoring error, never a guard
> read as *true*. A binding site of the format that does not take the list form MUST refuse a list
> with a message naming where the form is available, never read it as one comparison.

**Anchor:** Declarative glue > notifications - the sentence "`when:` supports a single
`field ==|!= literal` guard." is replaced by:

`when:` is the guard of the event axis: one `field ==|!= literal` comparison over the event record, or
a list of them meaning their AND (see *The event axis*). A status is named by its seeded name or its
id.

**Anchor:** Declarative glue > generates — create-from > Event-driven creation — `event:` (appended
after the first Normative paragraph).

The `when:` guard of an `onTransition` create-from MAY be a list: the mandatory status comparison plus
any number of `<StringField> ==|!= <literal>` comparisons over the source's own string fields, the
literal quoted or a bare word. A source whose status converges from an automatic path and a manual one
keeps the bare status guard on the document that must exist once regardless of path, and adds the
trace comparison only on the rules that record how the status was reached:

```yaml
generates:
  - name: log-driver-identified
    from: Fine
    to: FineLog
    event:
      onTransition: Fine
      mode: append
      when:
        - "Status == DRIVER_IDENTIFIED"
        - "resolution == found"        # the lookup's `outcome:` trace field
    map: { Fine: id }
```

> **Normative.**
> On an `onTransition` create-from a list guard MUST contain exactly one status comparison, an
> equality by seeded name or id; a list without it, or with a second numeric comparison, MUST be
> rejected. Every other element MUST compare one of the source's own `string` / `text` fields with a
> literal; a comparison against a field of another type, or against a property the source does not
> carry, MUST be rejected. The same property guarded twice MUST be rejected. All comparisons are
> evaluated against the source as re-read at delivery, as the single guard is. A generator SHOULD
> report - not reject - a literal comparison against a field a user can edit (one not `readOnly`),
> since an edit then changes which automations fire; a field the platform writes, such as a lookup's
> `outcome:` target, warrants no report.

**Anchor:** Processes & forms > processes > trigger - in the third bullet, "applies the `when` guard
(a single `field ==|!= literal`)" is replaced by:

applies the `when` guard - one `field ==|!= literal` comparison or a list of them meaning their AND,
a status named by its seeded name or its id -

**Anchor:** Data, seeds & naming > seeds > Status references — name, not number - the enumeration of
sites gains: the `event.when` guard of a `notifications` / `integrations` / `outbound` entry and of
an event-driven create-from, and a process trigger's `when`, each comparison of a list resolved on its
own.

## DSL index

| Construct | What it does |
| --- | --- |
| [`event.when`](#the-event-axis--lifecycle-events-and-process-step-events) | the guard of an axis binding: one `field ==\|!= literal` comparison over the event record, or a list of them meaning their AND - a status by seeded name or id |
