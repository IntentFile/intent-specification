# Where a field sits on the record page, or on no page - `placement: main | details | hidden`

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/153, https://github.com/eclipse-dirigible/dirigible/issues/8369, https://github.com/eclipse-dirigible/dirigible/issues/8370
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8384

## The problem

A record page has a main area, where the form's controls are, and a narrow details area beside it,
where read-only values are listed. A fixed rule decides which a field goes to: a read-only field to
the details area, everything else to the main area. A long read-only note then wraps over dozens of
lines in the narrow column beside a short form.

A technical column - a token, a correlation key that an integration writes - shows on the list, the
drawer and the details area, and no key keeps it off every page without making it a system column.
1.11 knows `readOnly` ("rendered read-only in the UI"), `major: false` ("keep the column off the
compact list table") and `visibleWhen` (a condition over the record), and none of them says where a
field sits, or that it sits nowhere.

## The proposed shape

```yaml
fields:
  - { name: notes, type: text,   readOnly: true, placement: main }   # long read-only text, main area
  - { name: token, type: string, placement: hidden }                 # on no generated page
```

## Expected behaviour

- `main` - a read-only field leaves the details area for the main area. It is shown as a value
  (its label, then the text with its line breaks kept), spanning the full row unless `size:` says
  otherwise. It shows when the record is viewed and edited, never as an input and never on the
  create form. On an editable field `main` changes nothing.
- `details` - the details area, the default for a read-only field.
- `hidden` - every generated page leaves the field out: the list, the drawer, the record page, the
  line editors. A create drops a value sent for it and an update keeps the stored value; every read
  still serves it, so a system writer and an integration can use it.

## Edge rules

- Refused when the intent is read: a value other than `main`, `details` or `hidden`; `details` on an
  editable field.
- `hidden` is refused on the primary key; on a `required` field with no server-side source (a
  `defaultValue` makes it acceptable); and together with `major: true`, `visibleWhen:`, a mention in
  the entity's `list:`, or a mention in a `related[].show` of another entity.
- A field without `placement:` renders as before; `details` and an absent key generate the same
  output.

## Prior art / workarounds

For the long note, nothing: it stayed in the narrow column. For the technical column, making it a
system column the application owns, which also took it out of the integration's reach, or accepting
it on every page. Reference implementation: https://github.com/eclipse-dirigible/dirigible/pull/8384.

## Specification text

**Anchor:** Entities & fields > fields, a row in the attribute table after `major: false`; and a new
subsection after Entities & fields > Control order.

Row in the fields table:

| Key | Meaning |
| --- | --- |
| `placement` | where the field sits on the record page - `main`, `details`, or `hidden` from every generated page (see [Placement on the record page](#placement-on-the-record-page)) |

New subsection:

#### Placement on the record page

A record page has a main area, where the form's controls are, and a details area beside it, where
read-only values are listed. By default a read-only field is listed in the details area and every
other field is a control of the main area. `placement:` overrides that for one field:

```yaml
fields:
  - { name: notes, type: text,   readOnly: true, placement: main }   # long read-only text, main area
  - { name: token, type: string, placement: hidden }                 # on no generated page
```

| Placement | Where the field sits |
| --- | --- |
| `main` | a read-only field is shown as a value in the main area, full width unless `size:` says otherwise |
| `details` | the details area - the default for a read-only field |
| `hidden` | no generated page; the API still serves it |

`main` is for read-only text too long for the details area: a note, a recorded reason, a message
received. `hidden` is for a column only a system writer or an integration reads and writes: a
token, a correlation key. Like `visibleWhen`, `hidden` is presentation and writes nothing away: the
stored value stays, and every read serves it.

> **Normative.**
> Under `placement: main` a conforming generator MUST render a read-only field in the main area of
> the record page as a value - its label, then the value with its line breaks kept - spanning the
> full row unless `size:` gives a width, when the record is viewed or edited; it MUST NOT render it
> as an input, nor on the create form, nor in the details area. On an editable field `main` changes
> nothing. Under `placement: hidden` it MUST leave the field out of every generated page - the
> lists, the drawers, the record page and the line editors; a create MUST drop a value sent for the
> field and an update MUST keep the stored value, while every read MUST still serve it. A field
> without `placement:`, or with `details`, MUST generate as it did before.

> **Normative.**
> Each of the following MUST be reported as an authoring error: a `placement` other than `main`,
> `details` or `hidden`; `details` on an editable field; `hidden` on the primary key, on a `required`
> field with no server-side source such as a `defaultValue`, together with `major: true` or
> `visibleWhen:`, and on a field the entity's `list:` or another entity's `related[].show` names.

## DSL index

| Construct | What it gives you |
| --- | --- |
| [`placement`](#placement-on-the-record-page) | where a field sits on the record page: `main` puts long read-only text in the main area, `details` the details area (the read-only default), `hidden` keeps a technical column off every generated page while the API still serves it |
