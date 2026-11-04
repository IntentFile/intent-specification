# A notify block takes cc and bcc, and a process step may send to a recipient a step worked out

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/132, https://github.com/eclipse-dirigible/dirigible/issues/7935, https://github.com/eclipse-dirigible/dirigible/issues/7938
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7937, https://github.com/eclipse-dirigible/dirigible/pull/7940

## The problem

A notify block has one recipient key, `to`. In 1.11 it "resolves a **literal** address, a **direct
field**, a **one-hop `relation.field`** of a to-one relation, or a **configuration reference**
`@config:KEY`". Two common needs have no declarative form:

- **A copy to someone who is not the addressee** - an archive mailbox, a support mailbox,
  accounting on every voided invoice, the archive on every dunning reminder. It has to be a second
  block, or hand-written mail code.
- **A list of recipients an earlier step worked out** - the approvers of a contract, the current
  owners of an account. No field of the record carries it, so the step cannot name it, and the only
  way to send is a hand-written sender that takes the message text out of the model and loses the
  step's `retry` / `onError`.

## The proposed shape

```yaml
processes:
  - name: contractReview
    trigger: { onCreate: Contract }
    vars:
      - { name: approverEmails }
    steps:
      - { name: findApprovers, kind: serviceTask, args: { delegate: ContractApprovers, produces: [approverEmails], next: tellThem } }
      - name: tellThem
        kind: serviceTask
        args:
          notify:
            to: "@var:approverEmails"              # a declared variable an earlier step set; may be a list
            cc: owner.email                        # a copy, in the forms `to` takes
            bcc: "@config:CONTRACT_ARCHIVE_EMAIL"  # in no delivered header
            subject: "Contract {number} needs your approval"
            body: "Please review it."
          uses: [approverEmails]
          next: done
      - { name: done, kind: end }

transitions:
  - name: void
    # ...
    notify:
      to: Customer.email
      cc: "@config:ACCOUNTING_EMAIL"               # copies work at every notify call site
      subject: "Invoice {number} was voided"
```

## Expected behaviour

Every notify block - a `notifications[]` entry, a `schedules[].notify`, a `transitions[].notify`
and a `serviceTask`'s `args.notify` - takes optional **`cc`** and **`bcc`** beside `to`, in the
forms `to` takes: a literal, a direct field, a one-hop `relation.field`, or `@config:KEY`. A value
may be a comma-separated list of addresses, and every address in it receives the message. A `bcc`
address appears in no header of the delivered message.

The message is sent when **any** of `to`, `cc` and `bcc` resolves to an address, so `to` may be
omitted and a `to` that resolves blank still lets the copies go out. When none of the three
resolves, the block is the no-op 1.11 already prescribes for a recipient that resolves to no
address: skipped and, with an `outcome`, stamped `skipped`. Inside a `forEach` fan-out, `cc` and
`bcc` resolve per row, as `to` does.

On a process step, a recipient - `to`, `cc` or `bcc` - may be **`@var:<name>`**: a process variable
an earlier step set, read from the process instance when the step runs. It depends on declared step
data - a process's `vars` and a step's `uses` - proposed in
https://github.com/IntentFile/intent-specification/issues/26; that half of this proposal is admitted
together with declared step data.

## Edge rules

- `cc` and `bcc` follow every rule `to` has: a multi-hop path and an empty `@config:` are rejected,
  a copy recipient that does not resolve drops the block as an unresolvable `to` does, and inside a
  fan-out a record-scoped (`record.`) copy recipient is rejected, as a record-scoped `to` is.
- `@var:<name>` is admitted on a process step's notify only. The name MUST be a variable the process
  declares and the step lists as one it uses; an undeclared or unlisted name, and an empty name, are
  rejected. On a transition, a schedule or a `notifications[]` entry it is rejected, in `to` and in
  the copy keys alike: nothing there runs inside a process instance.
- A misspelled key beside them (`ccc:`) is rejected as any unknown key of the block is.
- A block with neither key generates exactly as before.

## Prior art / workarounds

A second notify block for the copy, or a hand-written sender for a computed list. Reference
implementation: https://github.com/eclipse-dirigible/dirigible/pull/7937 (process steps, `@var:`)
and https://github.com/eclipse-dirigible/dirigible/pull/7940 (`cc` / `bcc` at every call site).

## Specification text

**Anchor:** Declarative glue > notifications, appended after the `> **Normative.**` blockquote on
`@config:` recipients ("... The form is admitted at every notify call site.").

A notify block may also name **`cc`** and **`bcc`** recipients, in the forms `to` takes, and any of
the three may be a comma-separated list. On a process step a recipient may be **`@var:<name>`**, a
declared process variable the step uses:

```yaml
notify:
  to: "@var:approverEmails"            # the approvers an earlier step worked out
  cc: owner.email                      # a copy, in the forms `to` takes
  bcc: "@config:ARCHIVE_EMAIL"         # a blind copy, named in no delivered header
  subject: "Contract {number} needs your approval"
```

> **Normative.** `cc` and `bcc` are OPTIONAL at every notify call site and MUST accept exactly the
> forms `to` accepts, under the same rejections, including the fan-out rule that a recipient MUST
> NOT be record-scoped. A recipient value MAY be a comma-separated list, and every address in it
> MUST receive the message; a `bcc` address MUST NOT appear in any header of the delivered message.
> A block MUST send when at least one of `to`, `cc` and `bcc` resolves to an address, and MUST be
> the no-op prescribed for a recipient that resolves to no address when none does. Inside a
> `forEach` fan-out, `cc` and `bcc` MUST resolve per row. A recipient `@var:<name>` is admitted on a
> process step's notify only, MUST name a variable the process declares and the step lists as used,
> and MUST be read from the process instance when the step runs; elsewhere, and with a name that is
> undeclared, unlisted or empty, it MUST be rejected at authoring time, the report naming the block.
> A block that declares neither `cc` nor `bcc` MUST behave as before.

The `@var:` sentence and its rule are folded together with declared step data (`vars` / `uses`,
https://github.com/IntentFile/intent-specification/issues/26); `cc` and `bcc` do not depend on it.

## DSL index

| Construct | What it does |
| --- | --- |
| [`notify.cc` / `notify.bcc`](#notifications) | copy recipients at every notify call site, in the forms `to` takes, each a list or one address; the block sends when any of the three resolves; `bcc` is named in no delivered header |
| [`to: "@var:<name>"`](#notifications) | a notify recipient read from a declared process variable the step uses, on a process step only |
