# A value required only under a condition

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7094,
  https://github.com/eclipse-dirigible/dirigible/issues/7237 (the guarded property's type)
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7129,
  https://github.com/eclipse-dirigible/dirigible/pull/7301

## The problem

`required: true` says a value must always be there. Most business rules about a missing value are
not like that: the value is needed for ONE way of handling the record and meaningless for the
others.

```yaml
- name: SalesInvoice
  fields:
    - { name: sentMethod, type: integer }   # 1 = e-mail, 2 = post, 3 = handed over
  relations:
    - { name: Customer, kind: manyToOne, to: Customer, required: true }
```

An invoice sent by e-mail needs the customer's e-mail address. One sent by post does not, and one
handed over needs neither. `required` on the customer's address would refuse every customer who is
never e-mailed, so it cannot be declared there; `checks:` has `exactlyOne`, `itemsSumEqual` and
`itemsMin`, none of which say it either. So nothing said it: an invoice with Sent Method = E-mail
whose customer carried no address went through Send, reached status SENT, and the mail step logged a
no-op for a recipient it did not have. Nothing was mailed, nothing was stamped on the record, and
the clerk who pressed the button was told it had succeeded.

The rule also has to bite where the value is finally NEEDED, not from the first draft. An invoice
being typed does not yet have a sent method, and a rule that fires on every save would refuse the
draft.

The same shape recurs across a suite: a bank transfer needs the counterparty's IBAN, a customs
declaration needs the consignee's tax number, a shipment by courier needs a delivery address, an
electronically filed return needs the signer's certificate id.

## The proposed shape

A row-level kind naming the value, the condition it is required under, and optionally the status at
which the condition is evaluated:

```yaml
- name: SalesInvoice
  checks:
    # the address lives on the related customer
    - { kind: requiredWhen, field: Customer.email, when: "sentMethod == 1", status: SENT,
        message: "Sent Method is E-mail but the customer has no e-mail address" }
    # ...and a rule about the record's own field, from the first save
    - { kind: requiredWhen, field: reference, when: "kind == 'export'",
        message: "An export needs a reference" }
```

- `field:` is the value that must be present: a field of the record, or a one-hop
  `Relation.field` over a to-one - the same path vocabulary a notification placeholder and a
  register lookup use, including a relation whose target is owned by another model.
- `when:` is the condition, one or more `<Property> ==|!= <literal>` comparisons over the record's
  own properties. A list is an implicit AND.
- `status:` is optional, and its presence is what decides WHEN the rule is evaluated.
- `message:` is what the person who wrote the record is told.

## Expected behaviour

- **Without `status:`** the rule holds on every user write, like `exactlyOne`: the write is refused
  with the authored message (400 over HTTP) on every surface the record can be written through.
- **With `status:`** the rule is evaluated when the record is persisted carrying that status - the
  transition that sends the document, not the drafting before it. The transition is refused with the
  authored message, and the refusal reaches whoever performed it.
- A value is "present" when it is neither absent nor blank. A record where the condition does not
  hold is unaffected.
- The value is read THROUGH the relation when the path names one: the related record is loaded and
  the field read from it. A relation that is not set is an absent value, so the check fires - which
  is the honest answer, since the value the rule is about cannot be reached.

## Edge rules

- A path may hop over at most one relation whose target is owned by another model, and it must be
  the last hop: what a foreign record points at in turn is known only to the model that owns it.
- The condition is read off the record itself - nothing is loaded to evaluate it - so every property
  it names must be the record's own field or to-one relation.
- The condition compares strings, integers of any width, booleans and a to-one's key: the types an
  equality is exact on. A decimal, a double or a date is refused rather than compared for equality,
  which is a question nobody means to ask of them. A property declared without a type is a string,
  as it is everywhere else, and is guardable as one; the type's spelling does not matter, and a
  refusal names the type as authored.
- A to-one's key compares **by value, whatever its width**. The key's width belongs to the target
  entity - and for a target owned by another model it is known only to that model - so a guard on a
  to-one holds for an `integer` key and a `long` key alike, rather than switching the rule off on a
  width the author never saw.
- A literal that is not a value of the compared property's type is refused. So is a condition that
  does not have the comparison shape at all: reading an uninterpretable condition as "always true"
  would turn the entry into an unconditional `required` nobody authored, and reading it as "always
  false" would switch the rule off - both silently.
- A status may be named by its seeded name rather than its id, as everywhere else a status is
  referenced. A `status:` gate requires the entity to have a status relation to read it from.
- Several `requiredWhen` entries on one entity are independent and all hold.

## Prior art / workarounds

Two, both worse than the rule they stand in for:

- A **hand-written step before the sending action** - a class that reads the record, a decision that
  branches on its answer, a hold task the record parks on, a form for that task, and a label for it:
  roughly fifteen declarative lines plus a class for one sentence of rule. It works, and it lands
  the clerk on a hold task instead of a refusal on the button they pressed.
- A **hand-written step after it**, which is worse: by then the document has been sent, and the
  refusal has nowhere to go but a background failure nobody is watching.
