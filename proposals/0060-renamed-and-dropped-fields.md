# A renamed field keeps its values, a retired one is dropped only on request

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7635
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7674

## The problem

An entity's table outlives the version of the file that created it. Once the application holds
data, the two most common changes to a field are not additions: a field is renamed, or a field is
retired. The file alone cannot tell them apart.

```yaml
# before
- name: Invoice
  fields:
    - { name: id,          type: integer, primaryKey: true, generated: true }
    - { name: invoiceDate, type: date }
    - { name: legacyCode,  type: string }

# after
- name: Invoice
  fields:
    - { name: id,        type: integer, primaryKey: true, generated: true }
    - { name: issueDate, type: date }
```

Read as two versions of one entity, the change says "`invoiceDate` disappeared, `issueDate`
appeared, `legacyCode` disappeared". Applied to a live table, that reading adds an empty
`issueDate` column beside the old one: every invoice already issued now shows no date, while its
date sits in a column nothing reads. Treating the disappearance of `legacyCode` as a removal is
worse in the other direction: a field taken out of the file by mistake - or merged away in a
conflict - deletes a column of production data on the next deployment, with no step anyone could
have reviewed.

Neither outcome is the author's intent, and the format has no way to state the intent.

## The proposed shape

```yaml
entities:
  - name: Invoice
    dropped: [legacyCode]
    fields:
      - { name: id, type: integer, primaryKey: true, generated: true }
      - { name: issueDate, type: date, renamedFrom: invoiceDate }
```

- `renamedFrom:` on a field names the field it was called before.
- `dropped:` on the entity lists former field and to-one relation names whose columns are to be
  removed, data included.

## Expected behaviour

- **A rename moves the values.** Applying the file renames the stored column, so the existing values
  are read under the new name instead of staying behind in the old column while the new field
  starts empty.
- **A rename is idempotent.** It acts only while the old column exists and the new one does not, so
  once the rename has happened it does nothing, and it may stay in the file.
- **A drop is explicit.** A name under `dropped:` has its column removed, values included. This is
  the contract step of an expand/contract change: first stop using a field (remove it from the file;
  its column and values are kept), then drop it once nothing reads it any more.
- **A deleted field is kept.** A field simply removed from the file keeps its column and its values.
  It is not dropped.

## Edge rules

All of the following are authoring errors:

- a `dropped:` name, or a `renamedFrom:` source, that the entity still declares as a field or a
  relation - compared by the column the name maps to, so a different spelling of a declared name
  is caught as well;
- a field `renamedFrom` its own name;
- a `renamedFrom:` source that is also listed under `dropped:`;
- two fields `renamedFrom` the same name;
- an empty name in either key.

## Prior art / workarounds

- A **hand-written migration** outside the model, run once per environment, which the model knows
  nothing about - the next regeneration from the file reintroduces the empty column on any
  environment the script did not reach.
- **Never renaming**: keeping a field's old name forever because renaming it would lose its data.
- Relational migration tools express the same two operations as explicit, reviewed steps (a column
  rename, a column drop), and the expand/contract pattern sequences them so a running application
  never reads a column that is gone.

## Specification text

**Anchor:** Entities & fields, three places:

1. *fields*, the key table: a new row after `translatable: false`:

| `renamedFrom` | the field's former name: applying the file renames the stored column, so the values keep up with the rename (see [renamedFrom / dropped](#renamedfrom--dropped--evolving-a-table-that-already-holds-data)) |

2. *Entity-level attributes*, the attribute table: a new row after `period:` / `immutableInPeriod:`:

| `dropped:` | former field / to-one relation names whose columns applying the file removes, data included (see [renamedFrom / dropped](#renamedfrom--dropped--evolving-a-table-that-already-holds-data)) |

3. A new subsection after *unique — a business key over more than one field* and before
   *defaultValue — field defaults*:

#### renamedFrom / dropped — evolving a table that already holds data

An entity's table outlives the version of the file that created it. Once the application holds
data, renaming a field and retiring one look alike from the file alone - a name disappears and,
for a rename, another appears - so the file says which it is:

```yaml
entities:
  - name: Invoice
    dropped: [legacyCode]
    fields:
      - { name: id, type: integer, primaryKey: true, generated: true }
      - { name: issueDate, type: date, renamedFrom: invoiceDate }
```

`renamedFrom:` on a field names the field it was called before. Applying the file renames the
stored column, so the existing values move with the new name instead of staying behind in the old
column while the new field starts empty. It acts only while the old column exists and the new one
does not, so it is harmless to leave in the file after the rename has run.

`dropped:` on an entity lists former field and to-one relation names whose columns applying the
file removes, data included. It is the explicit contract step of an expand/contract change: a field
is first removed from the file - its column and values are kept - and dropped once nothing reads it
any more.

A field simply deleted from the file is **not** dropped: its column and its values are kept.

> **Normative.**
> Applying a file MUST NOT remove the stored column, or the values, of a field or to-one relation
> that the file no longer declares, unless its name is listed under the entity's `dropped:`; a name
> listed there MUST have its column removed, values included. For a field declaring `renamedFrom:`,
> while the column of the former name exists and the field's own does not, applying the file MUST
> make the former column's values readable under the field's name; once either condition no longer
> holds, `renamedFrom:` MUST have no effect.
>
> Each of the following MUST be reported as an authoring error: a `dropped:` name or a
> `renamedFrom:` source that the entity still declares as a field or a relation, compared by the
> column the name maps to; a field `renamedFrom` its own name; a `renamedFrom:` source also listed
> under `dropped:`; two fields `renamedFrom` the same name; and an empty name in either key.

## DSL index

| Construct | What it does |
| --- | --- |
| [`renamedFrom`](#renamedfrom--dropped--evolving-a-table-that-already-holds-data) | a field's former name: the stored column is renamed in place, so the values move with the new name |
| [`entities.dropped`](#renamedfrom--dropped--evolving-a-table-that-already-holds-data) | former field / to-one relation names whose columns are removed, data included - a field merely deleted from the file keeps its column |
