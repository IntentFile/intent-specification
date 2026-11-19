# A record page says when the record or its latest related row needs attention - `banners`

- **Status:** released in [1.13](../versions/1.13.md)
- **Issue:** https://github.com/IntentFile/intent-specification/issues/158, https://github.com/eclipse-dirigible/dirigible/issues/8371
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8389

## The problem

A record page cannot draw attention to the record's state. An order whose latest shipment failed
opens like any other order: Edit is its filled button, and the failure - with the Retry that fixes
it - sits in the Shipments register at the bottom of the page. An order on hold says so only in its
status field. 1.11 can hide a field or a panel by state (`visibleWhen`), and nothing else on the page
reacts to it.

## The proposed shape

```yaml
entities:
  - name: Order
    related:
      - { entity: Shipment }
    banners:
      - text: "The last shipment failed: {failure}"   # required; {field} from the subject row
        variant: negative                              # information | warning | negative; default information
        related: { entity: Shipment, row: latest, when: "status == FAILED" }
        action: RetryShipment                          # optional: an action, transition or generates of the subject
      - text: "This order is on hold."
        variant: warning
        when: "status == ON_HOLD"                      # over the record itself
```

## Expected behaviour

- Each banner needs a condition: `when` over the record, `related: { entity, via?, row, when }` over
  one of the entity's own `related:` registers, or both - with both, both must hold.
- `row: latest` tests only the register's newest row, the one with the highest key; `row: any` holds
  when any row matches. A register without rows shows no banner.
- A banner that holds renders on the record page, while it previews the record, as an alert of its
  `variant`, between the task strip and the form.
- The text is translatable - seeded into the catalog - and its `{field}` placeholders are filled from
  the **subject** row: the record for `when`, the matched related row for `related`. A date
  placeholder is formatted as a date.
- `action` renders as an outlined button on the alert, offered only while the subject's actions
  offer it (its status guard and the caller's roles); running it evaluates the banners again, without
  a reload. The page's filled button is unchanged.
- The register's rows are read as its own panel reads them; a read that fails shows no banner.
- Banners decide what the page **says**, never what the server allows - like `visibleWhen`.

## Edge rules

- `text` is required; `variant` is `information`, `warning` or `negative`.
- `when` takes the `visibleWhen` grammar; status names resolve against the entity for `when` and
  against the register's source for `related.when`.
- The register is matched on `entity`, and on `via` where given. A match that is ambiguous without
  `via`, an `entity` that is no register of the entity and a cross-model register are refused.
  `row` must be `latest` or `any`, and `related.when` is required.
- A `{field}` must name one of the subject row's own fields; a relation or an unknown name is
  refused.
- `action` must be an action, transition or generates whose `forEntity` is the subject's entity.
- A model without `banners:` generates byte-identical output.

## Prior art / workarounds

None within the format: the reader had to scroll to the register and read its latest row to learn
the order needed attention. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8389.

## Specification text

**Anchor:** Entities & fields > Status-gated visibility - `visibleWhen`, as a new section after it.

### banners - the record page says what needs attention

`visibleWhen` hides what a state makes irrelevant. A banner says what a state makes urgent: a line
at the top of the record page, with the action that resolves it.

```yaml
- name: Order
  related:
    - { entity: Shipment }
  banners:
    - text: "The last shipment failed: {failure}"
      variant: negative
      related: { entity: Shipment, row: latest, when: "status == FAILED" }
      action: RetryShipment
    - text: "This order is on hold."
      variant: warning
      when: "status == ON_HOLD"
```

A banner holds when its condition does: `when` over the record, `related:` over one of the entity's
own [`related:`](#related--list-the-records-that-reference-this-entity) registers - its newest row
(`row: latest`, the highest key) or any of its rows (`row: any`) - or both, when both hold. The
`{field}` placeholders of its text are read from the row the condition tested: the record, or the
related row. Its `action`, an action, transition or create-from of that row's entity, is offered on
the banner while that row's actions offer it.

| Key | Meaning |
| --- | --- |
| `text` | the message, translatable; `{field}` placeholders name the subject row's own fields |
| `variant` | `information` (default), `warning` or `negative` |
| `when` | a condition over the record, in the `visibleWhen` grammar |
| `related` | `{ entity, via, row, when }` - a condition over a row of one of the entity's registers |
| `action` | an action, transition or `generates:` entry of the subject's entity, offered on the banner |

> **Normative.**
> A conforming generator MUST render each banner whose condition holds on the entity's record page
> while it previews the record, as an alert of the banner's `variant` placed between the pending-task
> strip and the form. A banner MUST declare `when`, `related` or both, and with both it holds only
> when both do. `related.row: latest` MUST test only the register's row with the highest key;
> `related.row: any` holds when any row satisfies `related.when`; a register without rows holds no
> banner, and a failed read of the register MUST show none. The text MUST be seeded into the
> default-language catalog and its placeholders filled from the subject row - the record for `when`,
> the tested related row for `related` - a date placeholder formatted as a date. An `action` MUST be
> rendered on the alert only while the subject's actions offer it to that row, and running it MUST
> evaluate the banners again without reloading the page; the page's filled button is not affected.
> A banner is presentation only: it MUST NOT change what the endpoint accepts.

> **Normative.**
> The following MUST be reported as authoring errors: a banner without `text`; a `variant` other
> than `information`, `warning` or `negative`; a banner with neither `when` nor `related`; a `when`
> or `related.when` that is not a valid `visibleWhen` condition, status names resolving against the
> entity for `when` and against the register's source for `related.when`; a `related.entity` that is
> not one of the entity's registers, a match that is ambiguous without `via`, and a cross-model
> register; a `related` without `when`, or with a `row` other than `latest` or `any`; a placeholder
> that is not a field of the subject row, a relation included; and an `action` that is not an
> action, transition or `generates:` entry whose `forEntity` is the subject's entity. A model without
> `banners:` MUST generate as it did before.

## DSL index

| Construct | What it does |
| --- | --- |
| [`banners`](#banners---the-record-page-says-what-needs-attention) | an alert on the record page while a condition over the record, or over its latest or any related row, holds - with the action that resolves it; presentation only |
