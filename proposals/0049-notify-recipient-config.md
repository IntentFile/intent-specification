# A notify recipient may be a configuration reference

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7385
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7387

## The problem

The recipient of a notify block is often not a property of any record. A staleness sweep reports to
the operations mailbox; a failure notice goes to whoever runs the deployment; a nightly summary goes
to "us". That address differs per environment - development, staging, each tenant's production - and
it is exactly the kind of value the format already keeps out of the source with `@config:KEY` on an
integration's `url:` and on a declared payload value.

A notify block's `to:`, however, admits only a literal, a direct field, or a one-hop
`relation.field`. Written in a recipient, `@config:OPS_EMAIL` is neither resolved nor refused: it
contains an `@`, so it reads as a literal address, the message is addressed to the eleven characters
of the key, and the only trace is the delivery failure. The author's choice is a hard-coded address
per environment, which makes the model environment-specific.

## The proposed shape

No new key. The fourth value form the recipient rule was missing:

```yaml
schedules:
  - name: stuckOrders
    cron: "0 */5 * * * ?"
    entity: Order
    where: [ { field: Status, op: eq, value: 2 } ]
    notify:
      to: "@config:OPS_EMAIL"
      subject: "Order {id} has not moved"
```

The same value is admitted in every notify block - a `notifications[]` entry, a `schedules[].notify`,
a `transitions[].notify`, and a sending process step's `notify`.

## Expected behaviour

A conforming generator MUST read the recipient from the configuration key **at send time**, on every
send. The key is not resolved at generation: the same generated application, deployed against a
different configuration, mails a different address without being regenerated. Where configuration is
tenant-scoped, the value in force for the sending tenant is the one read.

A key that is unset when the message is sent is a recipient resolving to no address - the block's
existing no-op: the send is skipped and recorded, never an error. Inside a schedule's tick that row
counts against the tick's "no recipient" total; inside a fan-out that row is skipped and the
remaining rows are still served.

## Edge rules

- **The rule is the prefix.** A value is a configuration reference exactly when it begins with
  `@config:` - the same test an integration `url:` and a payload value apply. A literal that merely
  contains an `@` (`ops@example.com`, and any address holding the marker elsewhere) is still an
  address. Whitespace around the key is not part of it.
- **An empty key is refused at parse** (`to: "@config:"`): it would resolve to nothing in every
  environment, so the block could never mail anyone. The message names the block and says the key is
  empty.
- **A dotted key is a key.** `@config:OPS.EMAIL` is not a multi-hop `a.b.c` path and MUST NOT be
  refused as one; the multi-hop rule applies to field paths only.
- **Not a field path.** A configuration reference is never validated against the entity's fields or
  relations, and a cross-model schedule's unresolvable-reference scan MUST skip it - there is nothing
  in the model for it to resolve against.
- **Fan-out.** Inside a `forEach:` a configuration reference is neither a bare path nor a
  record-scoped one; it resolves the same address for every row, which is legitimate (every row's
  message goes to the operations mailbox).
- **The current text disagrees with the shipped behaviour** in one place: the *notifications*
  section of the current version states the recipient forms as "a literal, a direct field, or a one-hop
  `relation.field`" and, by that sentence, a `@config:` value is a literal. The reference
  implementation resolves it. The Specification text below replaces that sentence.

## Prior art / workarounds

The workaround is the literal address, changed per environment by editing the model, or a
`kind: setting` entity holding the address with the notify block reading it through a relation -
which forces every record to carry a relation to a row whose only purpose is to be a mailbox. The
`@config:` reference already exists for an integration's endpoint and for a payload value; the
reference implementation ships it for a recipient in PR #7387, resolving through the same
configuration lookup at send time.

## Specification text

**Anchor:** Declarative glue > notifications - replacing the sentence "`to` and every `{placeholder}`
resolve a literal, a direct field, or a one-hop `relation.field` of a to-one relation." with the
paragraph below; the surrounding text of the section stands.

`to` resolves a **literal** address, a **direct field**, a **one-hop `relation.field`** of a to-one
relation, or a **configuration reference** `@config:KEY` - the same reference an integration's `url:`
and a payload value take. Every `{placeholder}` resolves a literal, a direct field, or a one-hop
`relation.field`. `when:` supports a single `field ==|!= literal` guard. Multi-hop paths (`a.b.c`) are
rejected with a clear message.

```yaml
notify:
  to: "@config:OPS_EMAIL"          # the operations mailbox is the deployment's, not the record's
  subject: "Order {id} has not moved"
```

> **Normative.** A recipient beginning with `@config:` names a configuration key and MUST be read at
> **send time**, on every send, never resolved at generation. A key unset at send time is a recipient
> that resolves to no address, and the block is the no-op that rule already prescribes: skipped and
> recorded, never an error. An empty key (`@config:`) MUST be rejected at parse with a message naming
> the block. The prefix is the whole test: a literal that merely contains an `@` is an address, a
> dotted key is not a multi-hop path, and a configuration reference is never checked against the
> entity's fields or relations. The form is admitted at every notify call site.

## DSL index

| Construct | What it does |
| --- | --- |
| [`to: "@config:KEY"`](#notifications) | a notify recipient read from configuration at send time; an unset key is the no-recipient no-op, an empty key is refused |
