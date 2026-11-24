# A share link's life read from the record - `expiry:` under `share:`

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8620
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8896
- **Depends on:** [0144](0144-share-link.md) (`share:` on an entity), [0184](0184-notify-share-url.md) (`{shareUrl}` in a notify body)

## The problem

`share:` ([0144](0144-share-link.md)) gives every link of an entity one life: `expiresAfterDays`, a
single number. The same document type is addressed to different readers, and they do not need the
same life. An invoice to a private individual carries a name and a home address; an application
wants its link to expire by default, while a link to a business's invoice stays permanent. A short
`expiresAfterDays` for everybody makes every business link die too; a permanent one leaves the
individual's document reachable for ever. Nothing in the block can say "expire after N days when the
record matches a condition".

## The proposed shape

```yaml
entities:
  - name: SalesInvoice
    function: Document
    share:
      when: ["status != DRAFT", "status != CANCELLED"]
      copy: SalesInvoiceCopy
      roles: [DocumentShare]
      expiresAfterDays: 0                        # every other link: permanent
      expiry:                                    # optional - the first entry that holds wins
        - { when: "Customer.Kind == 2", days: 30 }   # the customer is a private individual
    relations:
      - { name: status,   kind: manyToOne, to: InvoiceStatus, function: EntityStatus }
      - { name: Customer, kind: manyToOne, to: Customer }
```

## Expected behaviour

- When a link is created, its life is the `days` of the first `expiry` entry whose `when` holds
  for the record at that moment; when none holds, or no `expiry` is declared, it is
  `expiresAfterDays`. `days: 0` is a permanent link.
- The life is decided once, when the link is created, from the record as it is stored - never from
  anything the request carries - and the link keeps it: a later change to the record does not
  lengthen or shorten a link already created.
- It is the same for every way a link is created: a user's explicit request and a notify block's
  `{shareUrl}` ([0184](0184-notify-share-url.md)) read the same entries the same way.
- The deployment's setting that shortens the life of new links shortens a life chosen by `expiry`
  exactly as it shortens `expiresAfterDays`, and gives a permanent one its life; it never lengthens
  one.

## Edge rules

- `expiry` is a non-empty list; each entry has exactly the keys `when` and `days`, both required.
- `when` is a condition in the grammar of a `forbidWhen` check's: one `<property> ==|!= <literal>`
  term or a list of them (all must hold), where the property is a field or to-one relation of the
  entity or a one-hop `<Relation>.<field>` over a to-one relation, whose target may belong to
  another model. A status is named by its seeded name as there; a nomenclature owned by another
  model is named by its numeric id.
- A term that does not resolve - an unknown relation or field, more than one hop, a relation that is
  not a to-one - and a literal that is not a value of the compared type are authoring errors.
- `days` is a whole number from 0 (permanent) to 3650, as `expiresAfterDays` is.
- `expiry` changes how long a link lives, never whether it may exist: the `when` of `share:`, the
  roles and the stored copy still decide that.

## Prior art / workarounds

One short `expiresAfterDays` for every link of the entity, or a link revoked by hand when the reader
is a private individual. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8896.

## Specification text

**Anchor:** Entities & fields > share — an issued document through a link
([0144](0144-share-link.md)): the example gains the `expiry` lines, the key table a row after
`expiresAfterDays`, the *Life by record* paragraph below follows the table, and the section's
Normative block gains the paragraph below after *Revocation and expiry*. The *Declaration* paragraph's last
sentences read "`expiresAfterDays`, when present, MUST be a whole number from 0 to 3650; omitted, it
is 0. `expiry`, when present, MUST follow *Life by record*." The notify placeholder subsection of
[0184](0184-notify-share-url.md) reads "the declared `expiresAfterDays` or the `expiry` entry the
record meets" where it reads "the declared `expiresAfterDays`".

Example lines, after `expiresAfterDays: 0` in the section's example:

```yaml
    expiry:                       # optional - the first entry the record meets sets the life
      - { when: "Customer.Kind == 2", days: 30 }
```

Table row:

| `expiry` | optional - a list of `{ when, days }`: a link created while the record meets an entry's `when` lives that entry's `days` (the first that holds); `expiresAfterDays` otherwise - see *Life by record* below |

Paragraph after the table:

**Life by record.** One life for every link of an entity is often wrong: the same invoice type is addressed to a
business, whose link may stay, and to a private individual, whose document carries a home address
and should not be reachable for ever. `expiry` reads the life from the record. Each entry pairs a
condition with a number of days; when a link is created, the first entry whose condition holds for
the record gives the link its life, and `expiresAfterDays` covers every record no entry matches.
The condition is written as a [`forbidWhen`](#kind-forbidwhen--a-write-refused-while-a-condition-holds)
condition is - the record's own field, or one hop away, such as the customer's kind. The life is
fixed when the link is created and stays with the link.

> **Normative.** *Life by record.* `expiry`, when present, MUST be a non-empty list whose every
> entry declares `when` and `days` and no other key. `when` MUST be a condition in the grammar of a
> `forbidWhen` check's `when`, read off the declaring entity, and is refused at parse time as that
> condition is; `days` MUST be a whole number from 0 to 3650. When a link is created - by a user's
> request or by a notify block - its life MUST be the `days` of the first entry, in declared order,
> whose every term holds for the stored record at that moment, and `expiresAfterDays` when none
> does; 0 is a permanent link. The life MUST NOT be taken from the request, and MUST NOT change
> once the link is created. A deployment setting that shortens the life of new links MUST apply to
> that life exactly as to `expiresAfterDays`, and MUST NOT lengthen it.

## DSL index

| Construct | What it does |
| --- | --- |
| [`share.expiry`](#share--an-issued-document-through-a-link) | the life of a share link read from the record it is created on - the first `{ when, days }` entry the record meets, `expiresAfterDays` otherwise |
