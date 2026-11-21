# An issued document handed to an outside reader through a link - `share:` on an entity

- **Status:** draft
- **Issue:**
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/issues/8613

## The problem

A document leaves a generated application only as a mail attachment (`notify` with
`attach: print`). The one link the format can produce, `{recordUrl}`, points into the application
and needs a signed-in user with access to the record: right for a colleague, useless for a customer
or a supplier. Yet a link is what travels through a messenger or a text message, what a mail body
can carry instead of a large attachment, and what a "view online" code printed on a document points
at.

Nothing in the format says which documents an outside reader may ever receive, and it has to be
said in the model. An application's document store also holds drafts, uploaded attachments,
inbound documents and personal data; "any stored file is reachable by whoever knows its identifier"
turns one leaked identifier into a breach and cannot be reviewed. The set of things an anonymous
reader may receive must be named per entity, visible in a diff of the intent.

## The proposed shape

```yaml
entities:
  - name: SalesInvoice
    function: Document
    share:
      when: "status == ISSUED"      # required - only a record in this state can get a link
      copy: SalesInvoiceCopy        # required - the function: Snapshot child whose stored file is served
      roles: [SalesInvoiceShare]    # required - who may create and revoke links; [] admits nobody
      expiresAfterDays: 0           # optional - 0 (the default) = permanent
    relations:
      - { name: status, kind: manyToOne, to: InvoiceStatus, function: EntityStatus }
  - name: SalesInvoiceItem
    function: DocumentItem
    relations:
      - { name: SalesInvoice, kind: manyToOne, to: SalesInvoice, composition: true, required: true }
  - name: SalesInvoiceCopy
    function: Snapshot
    relations:
      - { name: SalesInvoice, kind: manyToOne, to: SalesInvoice, composition: true, required: true }

permissions:
  - { role: SalesInvoiceShare }
```

## Expected behaviour

- An entity that declares `share:` lets a user holding one of the declared roles create a **share
  link** on a record that is in the `when` state and has a stored copy. Whoever holds the link opens
  the record's issued copy without an account.
- An entity that declares no `share:` has no links and no way to get one. Sharing is opt-in per
  entity; no deployment setting makes an undeclared entity shareable.
- A link is created only by a user's explicit act on the record. Issuing the document, or minting
  its copy, creates none.
- The link carries a random token that has nothing to do with any identifier of the record. The
  application stores only a hash of it and discloses the link once, when it is created.
- A link serves one stored file, fixed when the link is created: the record's newest `copy` entry
  at that moment. Opening the link serves that file and nothing else - never the live record, a
  new render, another file of the record or a listing. A copy minted later, after an amendment,
  is reached only through a new link.
- A stored PDF is displayed inline; a stored file of any other type is sent as a download.
- A token that does not answer - unknown, expired, revoked, created in another tenant, or whose
  file is gone - gets one and the same not-found answer.
- A link can be revoked by a user holding one of the roles, with immediate effect. A link may
  expire, `expiresAfterDays` after it was created.
- A record that leaves the `when` state loses its links for good, whoever moved it - a user, a
  process step, a schedule, a delete. Bringing the record back into the state revives none of them.
- A link records who created it and when, who revoked it and when, how often it was opened and
  when it was last opened. Every opening is recorded with its time, the link, the remote address
  and the user agent.
- A deployment may let a tenant switch sharing off and shorten the life of new links. Neither
  setting can enable sharing for an entity that does not declare it or lengthen a declared life.

## Edge rules

- `when`, `copy` and `roles` are required keys. A `share:` without a `when` would make a draft as
  shareable as an issued document.
- `when` is a record condition in the grammar of `visibleWhen` (`<property> ==|!= <literal>`, a
  status by its seeded name) and must guard a field or to-one relation of the entity.
- `copy` must name an entity of the same model that is a `function: Snapshot` composition child of
  the declaring entity, and the declaring entity must be a printable document (it has a
  `function: DocumentItem` child) - otherwise no copy is ever minted.
- `roles` lists roles declared under `permissions`; a role the model does not declare is an
  authoring error. `roles: []` is legal and admits nobody, the administrator included.
- `expiresAfterDays` is a whole number of days from 0 (permanent) to 3650; omitted, it is 0.
- `share:` on a **personal** entity - one that carries a `personal: true` relation, or whose
  composition ancestor does - is an authoring error. A link is a bearer credential: whoever holds it
  reads the document. A personal document stays behind sign-in; a notification about it carries
  `{recordUrl}`.
- An unknown key under `share:` is an authoring error, as everywhere.
- Not part of this proposal: a notify placeholder resolving to a share link, payment or acceptance
  on the page a link opens, a second factor on a link, sharing a report or an arbitrary file.

## Prior art / workarounds

Sending the document as a mail attachment, or a hand-written public endpoint outside the model -
which nobody can review from the intent. Hosted-document links are an established pattern of
invoicing products. Reference implementation: tracked in
https://github.com/eclipse-dirigible/dirigible/issues/8613.

## Specification text

**Anchor:** Entities & fields > Entity-level attributes: a table row after `duplicable:`; and the
section below, placed after "Attachments and snapshots".

Table row:

| `share:` | lets a user hand the issued copy of a record to a reader without an account, through a revocable link (see [share — an issued document through a link](#share--an-issued-document-through-a-link)) |

### share — an issued document through a link

A customer or a supplier has no account in the application, and a document they must read cannot
always travel as an attachment. `share:` on a document entity declares that the **issued copy** of
its records may be opened through a link a user creates on the record and sends on:

```yaml
- name: SalesInvoice
  function: Document
  share:
    when: "status == ISSUED"      # required - only a record in this state can get a link
    copy: SalesInvoiceCopy        # required - the function: Snapshot child whose stored file is served
    roles: [SalesInvoiceShare]    # required - who may create and revoke links; [] admits nobody
    expiresAfterDays: 0           # optional - 0 (the default) = permanent
```

| Key | Meaning |
| --- | --- |
| `when` | required - a record condition in the grammar of [`visibleWhen`](#status-gated-visibility--visiblewhen); only a record in this state can have a link |
| `copy` | required - the [`function: Snapshot`](#attachments-and-snapshots) composition child of this entity whose stored file a link serves |
| `roles` | required - the roles, declared under [`permissions`](#permissions), that may create and revoke links; an empty list admits nobody |
| `expiresAfterDays` | optional - the life of a new link in whole days, 0 to 3650; 0, the default, is permanent |

Sharing is declared, never configured: the entities whose documents an anonymous reader may ever
receive are exactly those that carry a `share:` block, so the question is answered by reading the
intent. A link is a bearer credential - whoever holds it reads the document - which is why a
[personal](#personal-and-partner-surfaces) entity cannot declare one.

> **Normative.**
> **Declaration.** `share:` MUST declare `when`, `copy` and `roles`; a block missing any of them is
> an authoring error reported at parse time. `when` MUST be a record condition that guards a field
> or to-one relation of the declaring entity. `copy` MUST name an entity of the same model that
> declares `function: Snapshot` and is a composition child of the declaring entity, and the
> declaring entity MUST be a printable document; any other `copy` is an authoring error. Every
> entry of `roles` MUST be a role declared under `permissions`. `expiresAfterDays`, when present,
> MUST be a whole number from 0 to 3650; omitted, it is 0. `share:` on an entity that carries a
> `personal: true` relation, or whose composition ancestor carries one, MUST be refused at parse
> time.
>
> **Opt-in.** A conforming generator MUST NOT make any file of an entity that declares no `share:`
> reachable through a share link, and MUST NOT offer a setting that does. A deployment MAY let a
> tenant disable sharing or shorten the life of new links; such a setting MUST NOT enable sharing
> for an undeclared entity or lengthen a declared life.
>
> **Creating a link.** A link MUST be created only by an explicit request of an authenticated
> caller holding one of the declared `roles`, on a record that is in the `when` state and has a
> stored copy; any other request MUST be refused. An empty `roles` list MUST admit no caller,
> whatever other roles the caller holds. Issuing a document or minting its copy MUST NOT create a
> link.
>
> **The token.** A link MUST carry a token of at least 128 bits drawn from a cryptographically
> secure random source, independent of every identifier of the record. Only a hash of the token
> MUST be stored, and the link MUST be disclosed once, in the answer that creates it.
>
> **Opening a link.** A link is opened at an unauthenticated URL carrying the token. The file a
> link serves MUST be fixed when the link is created and MUST be an immutable stored copy - the
> record's newest `copy` entry at that moment; a copy minted later is served only through a new
> link. The answer MUST be that file and nothing else - never the live record, a new render,
> another file or a listing. A PDF SHOULD be displayed inline, and a stored file of any other type
> SHOULD be sent as a download. A token MUST resolve only within the tenant its
> link was created in. A token that is unknown, expired, revoked, presented in another tenant, or
> whose file no longer exists MUST receive one and the same not-found answer, indistinguishable
> between those cases. Opening a link MUST NOT read or create an authenticated session and MUST
> NOT change any record, process or task. The answer SHOULD forbid caching and indexing and
> SHOULD withhold the referrer.
>
> **Revocation and expiry.** A caller holding one of the declared `roles` MUST be able to revoke a
> link, and a revoked link MUST stop answering immediately, on every node of a deployment and
> across restarts. A link whose life has elapsed MUST stop answering; expiry MUST be enforced by
> the server.
>
> **Leaving the state.** A link of a record that is no longer in the `when` state MUST NOT answer,
> whatever moved the record out of it, and MUST be revoked permanently: a record that returns to
> the state MUST NOT regain it.
>
> **Audit and throttling.** An implementation MUST record who created a link and when, who
> revoked it and when, and each opening of it with its time, the link, the remote address and the
> user agent. The record's users
> holding one of the `roles` MUST be able to see each link's state, expiry, number of openings and
> last opening. The unauthenticated URL MUST be rate limited per remote address and per link, and
> an address that repeatedly presents tokens that do not answer MUST be refused for a time.

## DSL index

| Construct | What it does |
| --- | --- |
| [`share` (entity)](#share--an-issued-document-through-a-link) | lets a user hand a record's issued copy to a reader without an account through a random, revocable, optionally expiring link; opt-in per entity, refused on a personal entity |
