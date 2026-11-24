# A share link in a message - `{shareUrl}` in a notify body

- **Status:** proposed
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8616
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8895
- **Depends on:** [0144](0144-share-link.md) (`share:` on an entity)

## The problem

`share:` ([0144](0144-share-link.md)) lets a user create a link to the issued copy of a document and
send it on by hand. The most common way that link should travel is the mail the application already
sends: "mail the invoice" with a link instead of, or next to, a multi-megabyte attachment. A notify
block cannot say so. Its only link placeholder, `{recordUrl}`, opens the record inside the
application and needs an account the customer does not have; an unreserved `{shareUrl}` resolves as
a field of the record and fails.

Three things keep it from being a one-line addition. A link serves the stored copy, so it can only
be created after the copy is minted - and a send bound to the step that issues the document often
runs first. A link created by a send is created by the application, not by a user holding one of
the `share: roles`, so the authority needs a rule of its own. And a send whose link cannot be
created must not go out with a blank where the link should be.

## The proposed shape

No new key. A reserved placeholder, legal in the body of a notify block about an entity that
declares `share:`:

```yaml
processes:
  - name: InvoiceIssue
    trigger: { onCreate: SalesInvoice }
    steps:
      - { name: mint, kind: serviceTask, args: { delegate: SalesInvoiceSnapshotGenerator, next: mail } }
      - name: mail
        kind: serviceTask
        args:
          notify:
            to: Customer.email
            subject: "Invoice {number}"
            body: "Read it online: {shareUrl}"
            outcome: mailOutcome
          next: end
```

## Expected behaviour

- `{shareUrl}` in a `body` or `html` resolves to the absolute URL of a share link to the issued copy
  of the record the message is about.
- The link is created when the message is sent, after its recipient is known, under the same rules
  as a link a user creates: the record is in the `share: when` state, its copy is stored, the
  deployment allows sharing for the tenant, and the link lives `expiresAfterDays`. A message with
  nobody to send to creates no link.
- One link is created per message, recorded as created by the application with the sending block
  named, and listed on the record with the links users created; it can be revoked like any of them.
- The link's address is built on the application's configured external address, never on anything
  a request supplied.
- When no link can be created - the record is outside the `when` state, no copy is stored yet, or
  sharing is switched off - the message is not sent. The block's delivery contract applies as for
  any failed delivery: its `outcome:` field records the failure and why, `onNotifyFailed` is
  announced, and a process step fails under its `retry:` / `onError:`.

## Edge rules

- `{shareUrl}` on a notify block whose record's entity declares no `share:` is an authoring error.
  It is the same opt-in a user's link needs: no message can hand out a document its entity does not
  declare shareable.
- `{shareUrl}` in a `subject` is an authoring error: a link is a bearer credential, and a subject is
  shown in every mail list and preview. It is legal in `body` and `html`.
- `{shareUrl}` inside a `forEach` fan-out is an authoring error: each message there is about a row,
  while the document belongs to the record the rows hang off.
- `{shareUrl}` on a `schedules[].notify` whose source is cross-model is an authoring error, as
  `{recordUrl}` is: the record's `share:` and the code that creates its links belong to the model
  that owns it.
- `shareUrl` joins `recordUrl`, `inboxUrl` and `appUrl` as a reserved name; an entity field of the
  same name does not shadow it.
- Ordering is the author's: bind the block to a step after the one that mints the copy, or to a
  transition that follows it. A block that runs earlier fails as described above, every time.

## Notes

Not part of this proposal: the anchor record's link inside a fan-out (one link per recipient to the
document they were all sent), and an expiry read from the record.

## Prior art / workarounds

Before this, the link was created by hand on the record page and pasted into a mail, or a
hand-written listener called the share facade. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/issues/8616.

## Specification text

**Anchor:** Declarative glue > The notify block > Links back to the application: `{recordUrl}`,
`{inboxUrl}`, `{appUrl}` - a fourth table row, and the subsection below placed after the section's
last Normative paragraph. The release also amends the *Creating a link* rule of
[share — an issued document through a link](#share--an-issued-document-through-a-link): "A link MUST
be created only by an explicit request of an authenticated caller holding one of the declared
`roles`, or by a notify block of the model that carries `{shareUrl}`, ..."; the share section's last
edge rule ("Not part of this proposal: a notify placeholder resolving to a share link, ...") drops
its first item; and the placeholders section's first Normative paragraph opens "The four names are
RESERVED at every notify call site".

Table row:

| `{shareUrl}` | a new share link to the issued copy of the record the message is about - see [share](#share--an-issued-document-through-a-link); `body` and `html` only |

##### A share link in the message: `{shareUrl}`

`{shareUrl}` is for the reader without an account: the customer a document is mailed to receives a
link that opens its issued copy, where `{recordUrl}` would ask them to sign in. The link is created
when the message is sent, under the rules of a link a user creates, and is listed on the record, as
created by the application, with the links users created. Bind the block to a step after the one
that mints the copy, or to a transition that follows it: before the copy exists no link can be made,
and the message is not sent.

> **Normative.** `{shareUrl}` MUST be refused at parse time on a notify block whose record's entity
> declares no `share:`, in a `subject`, inside a `forEach`, and on a `schedules[].notify` whose source
> is cross-model. When the message is sent, after its recipient is resolved, the implementation MUST
> create one share link to the record's issued copy under the rules a user's link is created under -
> the `when` state, a stored copy, the tenant's switch and its cap, the declared `expiresAfterDays` -
> record the application as its creator with the sending block named, and put the link's absolute
> URL, built on the configured external address of the application, in its place. When no link can
> be created the message MUST NOT be sent, and the failure MUST follow the block's delivery
> contract - `outcome:` and `onNotifyFailed` as for a failed delivery. A message with nobody to
> send to MUST NOT create a link. An implementation MUST NOT log the URL.

## DSL index

| Construct | What it does |
| --- | --- |
| [`{shareUrl}`](#a-share-link-in-the-message-shareurl) | a share link to the issued copy of the record, created when the message is sent - for the reader without an account |
