# A list may show a record's child rows and its audit columns; a register may show its source's audit columns

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/164, https://github.com/eclipse-dirigible/dirigible/issues/8295, https://github.com/eclipse-dirigible/dirigible/issues/8376, https://github.com/eclipse-dirigible/dirigible/issues/8062
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8303, https://github.com/eclipse-dirigible/dirigible/pull/8377, https://github.com/eclipse-dirigible/dirigible/pull/8063

## The problem

Three gaps in what a list column may name, all in text 1.11 already has.

1. **A record's children are invisible in every list.** Proposal 0065's rule reads "Each name MUST
   be a field, a to-one relation or the entity's stored display name; a to-many relation ... MUST be
   reported as authoring errors". So a reader opens each candidate to learn which skills the
   candidate holds, and each team member to learn what was granted.
2. **An audited entity cannot list who created a row and when.** `audit: true` "adds the four
   standard audit columns, populated automatically", yet `list:` refuses `CreatedAt`, `CreatedBy`,
   `UpdatedAt` and `UpdatedBy` as unknown names: they are neither fields nor relations.
3. **A register cannot show them either.** The `related` section says "Every `show:` name MUST be a
   field or relation of the referencing entity", and the audit columns are neither - so a customer's
   support-ticket register cannot say when each ticket was opened and by whom.

## The proposed shape

```yaml
entities:
  - name: Candidate
    audit: true
    list: [name, skills.skill, CreatedAt]     # a one-hop collection path, and an audit column
    relations:
      - { name: skills, kind: oneToMany, to: CandidateSkill, composition: true, label: Skills }
  - name: CandidateSkill
    relations:
      - { name: candidate, kind: manyToOne, to: Candidate, composition: true }
      - { name: skill,     kind: manyToOne, to: Skill }
  - name: Customer
    related:
      - { entity: SupportTicket, via: customer, show: [subject, CreatedAt, CreatedBy] }
  - name: SupportTicket
    audit: true                               # the register's source must be audited
```

## Expected behaviour

**A collection path.** A `list:` entry may be `<collection>.<property>` - a `oneToMany` of the
entity and a property of its child - or a bare `<collection>`, short for `<collection>.Name`, which
needs the child's entity-level `label:`. A `manyToMany` is reached as the `oneToMany` to its link
entity.

- The column renders each row's children as a set of short labels (chips), one per child: the
  property's value, or for a to-one property the target's display text - read through the
  translation overlay where the target is multilingual, as its picker shows it. The children come
  in the child's `orderBy:` (proposal 0086), else in its key order; a row without children shows
  none.
- The children are read once per page of rows, never once per row. The column is filled on the
  list read, on search and on the single-record read.
- The column is headed by the collection's `label:`. A `label:` on a collection, refused by 1.11 as
  rendering no control, is accepted once the list shows the collection.
- The column is not sortable and is not offered as a filter. An export writes the children's texts
  joined by a comma.
- The column is read-only: a write that carries it changes nothing.
- The column appears in every list that follows `list:` - the entity's own list, a composition
  child's detail panel, a `related:` register without `show:`. The personal and partner lists and
  the form or document summary lines leave it out.

**The audit columns.** For an `audit: true` entity, `CreatedAt`, `CreatedBy`, `UpdatedAt` and
`UpdatedBy` (case-insensitive) are valid `list:` names. For an `audit: true` register source they
are valid `show:` names. The `At` pair renders as a date-time, the `By` pair as text.

## Edge rules

Refused when the intent is read, each with a message naming the token:

- an unknown collection, or an unknown property of the child;
- a path deeper than one hop;
- a property that is itself a collection;
- a bare collection whose child declares no `label:`;
- a child that does not reference the entity through exactly one to-one relation;
- a collection column whose name collides with a declared property;
- an audit name in `list:` of an entity that is not `audit: true`, or in `show:` of a source that is
  not `audit: true`, with a message naming the flag.

A cross-model register (`model:`) is not validated against the referencing model, which cannot see
the source; a generator resolves it against the owner model, where an audited source carries the
audit columns. A model that names no collection path and no audit column generates as before.

## Prior art / workarounds

None within the format. The reader opened each record to see its children, and the audit
columns, though stored and returned by the endpoint, could not be named in a list or a register.
Reference implementations:
https://github.com/eclipse-dirigible/dirigible/pull/8303 (collection path),
https://github.com/eclipse-dirigible/dirigible/pull/8377 (audit columns in `list:`),
https://github.com/eclipse-dirigible/dirigible/pull/8063 (audit columns in `show:`).

## Specification text

**Anchor:** Entities & fields > Control order, appended after the `list:` rules (amends proposal
0065's normative block); and Relations & multi-model > related - list the records that reference this
entity, whose `show:` sentence is replaced.

*In Control order, after the `list:` normative block:*

A list may also show what a row **holds**. A `list:` entry may name a one-hop path into a
`oneToMany` collection - `<collection>.<property>` - or the collection alone, which shows each
child's stored display name:

```yaml
- name: Candidate
  audit: true
  list: [name, skills.skill, CreatedAt]
  relations:
    - { name: skills, kind: oneToMany, to: CandidateSkill, composition: true, label: Skills }
```

The column shows each row's children as short labels, one per child, in the child's `orderBy:` or
else its key order; a to-one property shows its target's display text. It is headed by the
collection's `label:` and is read-only, unsortable and not offered as a filter. A `manyToMany` is
reached as the `oneToMany` to its link entity. On an `audit: true` entity the four audit columns are
list names like any field.

> **Normative.**
> This amends the rule above: a `list:` name MAY also be `<collection>.<property>`, where
> `<collection>` is a `oneToMany` relation of the entity and `<property>` is a field or a to-one
> relation of its child, or a bare `<collection>`, which stands for the child's stored display name
> and requires the child to declare `label:`. A conforming generator MUST render such a column as the
> set of the row's children's values - a to-one property by its target's display text, read through
> the target's translation overlay where the target is `multilingual: true` - ordered by the child's
> `orderBy:` or else by the child's key. It MUST read the children of a page of rows together, never
> once per row; MUST head the column with the collection's `label:`, which this makes valid on a
> collection; MUST NOT offer the column as a sort or a filter; MUST write it to an export as the
> values joined by a comma; and MUST ignore it on a write. The column is part of every list governed
> by `list:` - the entity's list, a composition child's detail panel and a `related:` register
> without `show:` - and is left out of the personal and partner surfaces. An unknown collection or
> property, a path of more than one hop, a property that is itself a collection, a bare collection
> whose child declares no `label:`, a child that does not reference the entity through exactly one
> to-one relation, and a column name that collides with a declared property MUST be reported as
> authoring errors naming the entry.

> **Normative.**
> On an `audit: true` entity, `CreatedAt`, `CreatedBy`, `UpdatedAt` and `UpdatedBy` MUST be accepted
> as `list:` names, matched case-insensitively; the `At` pair renders as a date-time and the `By`
> pair as text. On any other entity they MUST be reported as authoring errors whose message names
> the missing `audit: true`. A model that names no collection path and no audit column MUST generate
> as it did before.

*In related, replacing the sentence "Every `show:` name MUST be a field or relation of the
referencing entity.":*

> Every `show:` name MUST be a field or relation of the referencing entity or, when that entity is
> `audit: true`, one of its four audit columns, matched case-insensitively. An audit column named on
> a source that is not `audit: true` MUST be reported as an authoring error whose message names the
> missing flag. The `show:` names of a cross-model register are resolved against the owner model, as
> its columns are.

## DSL index

No new row; the `list` row gains "a one-hop collection path rendered as the children's labels, and
the audit columns of an audited entity", and the `related` row gains "whose `show:` may name an
audited source's audit columns".
