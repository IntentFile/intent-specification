# A roll-up reads the columns a file child and an audited child carry

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7947
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8887

## The problem

A `function: Attachment` or `function: Snapshot` child carries the metadata of its file - its name,
content type, size, storage path and folder key - and, like an `audit: true` entity, who created
and changed each row and when. None of these columns is declared by the author; the child carries
them by being a file child. A roll-up cannot read them: its `of`, `by` and `where` are checked
against the declared fields only, so a model that names one is refused.

The case is a company's logo on its invoices. The logo is uploaded onto the company record as an
attachment child. A print template reaches a related record one hop away
(`{{document.Company.<field>}}`), but the logo row is a second hop from the invoice. The natural
bridge is a roll-up that copies the newest upload's storage path onto the company, ordered by
when each row was created - and both of those are columns the author never declares.

## The proposed shape

```yaml
entities:
  - name: Company
    fields:
      - { name: logo,      type: string, length: 1000 }
      - { name: pngImages, type: integer }
  - name: CompanyLogo
    function: Attachment
    relations:
      - { name: Company, kind: manyToOne, to: Company, composition: true, required: true }
rollups:
  - { name: latestLogo, entity: CompanyLogo, via: Company, field: logo,
      op: latest, of: StoragePath, by: CreatedAt }
  - { name: pngImages, entity: CompanyLogo, via: Company, field: pngImages,
      where: [ { field: ContentType, op: eq, value: image/png } ] }
```

## Expected behaviour

- **The carried columns are readable.** A roll-up's `of`, `by` and `where` may name a column the
  child carries without declaring it, exactly as they name a declared field.
- **They keep their types.** `FileSize` is a whole number, `CreatedAt` / `UpdatedAt` are
  timestamps, a snapshot's `Version` is an integer and the others are text, so every type rule a
  roll-up already has applies to them unchanged.
- **The case works.** In the example, each upload moves the company's `logo` to the newest file's
  storage path, deleting that upload brings the previous one back, and `pngImages` counts only the
  uploads whose content type is `image/png`.

## Edge rules

- A child that is neither a file child nor `audit: true` carries none of these columns, and naming
  one is still an authoring error.
- An `audit: true` entity that is not a file child carries the audit columns only.
- The names match case-insensitively, as `list:` matches the audit columns.

## Prior art / workarounds

The file is placed by hand at a fixed path in the document store per company, outside the model.

## Specification text

**Anchor:** Declarative glue > rollups, after the `where:` / `guardAt:` / `message:` paragraph and
its normative blocks.

With `op: latest`, the roll-up keeps `field` equal to the `of` value of the child with the greatest
`by` - a date or timestamp of the child - and has no value when the parent has no children.

A roll-up reads the columns its child carries as well as those the child declares: a
`function: Attachment` or `function: Snapshot` child's `FileName`, `ContentType`, `FileSize`,
`StoragePath` and `Uuid` (and a snapshot's `Version`), and the `CreatedAt`, `CreatedBy`,
`UpdatedAt` and `UpdatedBy` of a file child or an `audit: true` entity. The newest upload mirrored
onto its parent:

```yaml
rollups:
  - { name: latestLogo, entity: CompanyLogo, via: Company, field: logo,
      op: latest, of: StoragePath, by: CreatedAt }
```

> **Normative.**
> A roll-up's `of`, `by` and `where` MUST accept, matched case-insensitively, the columns its child
> carries: on a `function: Attachment` or `function: Snapshot` child `FileName`, `ContentType`,
> `FileSize`, `StoragePath`, `Uuid` and the four audit columns, on a snapshot also `Version`, and on
> an `audit: true` entity the four audit columns. They MUST be typed as `FileSize` a whole number,
> `CreatedAt` / `UpdatedAt` timestamps, `Version` an integer and the others text, and every type
> rule of the roll-up MUST apply to them. On any other child they MUST be reported as authoring
> errors. With `op: latest`, `of` and `by` are required, `by` MUST be a date or timestamp, `field`
> MUST have the type of `of`, and a parent with no children MUST have no value in `field`.

## DSL index

| Construct | What it does |
| --- | --- |
| [`rollups[].op: latest`](#rollups--denormalised-parent-totals) | keeps `field` equal to the `of` value of the child with the greatest `by` date or timestamp |
| [`rollups[].of` / `by` / `where`](#rollups--denormalised-parent-totals) | may also name the columns a file child or an `audit: true` child carries - its file metadata and its audit columns |
