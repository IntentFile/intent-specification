# exports - a fixed-width, delimited or XML file over a query, every run recorded on a ledger

- **Status:** released in [1.13](../versions/1.13.md)
- **Issue:** https://github.com/IntentFile/intent-specification/issues/141, https://github.com/eclipse-dirigible/dirigible/issues/7855, https://github.com/eclipse-dirigible/dirigible/issues/7925
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7885, https://github.com/eclipse-dirigible/dirigible/pull/7999

## The problem

Every statutory filing - a VAT ledger, a payroll declaration, a standard audit file, an intrastat
return - and every ledger feed to a third party is "a file in an exacting format over a query".
1.11 has no construct for it: each such file was hand-written code per format per country, and each
one decided on its own whether a value too wide for its column is cut, padded or refused.

A regulator's format also often accepts far fewer characters than the master data holds - uppercase
Cyrillic, digits and `-` in the names of one declaration; Latin, `№` and `/` in another of the same
agency - while the registered name must stay as registered. The normalisation was therefore a helper
per file too, and a name that lost its letters on the way into the file was written without anyone
noticing.

## The proposed shape

```yaml
exports:
  - name: SupplierDeclaration
    source: PurchaseInvoice              # an entity, or a plain report
    format: fixedWidth                   # fixedWidth | delimited | xml
    encoding: windows-1251
    parameters: [{ field: issuedOn, op: between }]   # an entity filter, or the report's own parameters
    orderBy: [issuedOn, number]
    fileName: "DECL_{from}_{to}.txt"     # {param} tokens
    ledger: ExportRun                    # an entity declared with no fields
    header:
      - { literal: "HDR", width: 3 }
      - { param: from, width: 8, pattern: yyyyMMdd }
    line:
      - { from: supplier.vatNumber, width: 15, align: left, pad: " " }
      - { from: supplier.name, width: 60, case: upper, allowed: "А-Я0-9 -", onDisallowed: strip,
          collapseSpaces: true, onLoss: fail, replace: { "ЕООД": "", "ООД": "" } }
      - { from: total, width: 12, align: right, pad: "0", scale: 2, decimalSeparator: "." }
    trailer:
      - { from: "count()", width: 6 }
      - { from: "sum(total)", width: 14, scale: 2 }

entities:
  - name: ExportRun                      # its columns are supplied by the export
```

A model without `exports` generates exactly as in 1.11.

## Expected behaviour

**The file.** Records are `header`, `line` (one per source row) and `trailer`. A field reads exactly
one of `from:` (a field, a one-hop `relation.field`, a report dimension or measure, `sum(...)` or
`count()`), `literal:` or `param:`. Its layout is `width`, `align`, `pad`, `truncate`, `scale`,
`decimalSeparator`, `pattern`, and for XML `name` and `attribute`. The generated application serves
a **download** and an entry point a scheduled job or hand-written code can call, open to the roles
granted the export's `read` permission.

**Nothing is written approximately.** A value wider than its field without `truncate`, a character
the encoding cannot represent, or a character XML does not allow fails the run: it is recorded
FAILED and answered **422**. A missing or unreadable parameter is answered **400** and records
nothing.

**The ledger.** `ledger:` names an entity the author declares with no fields. The export supplies its
columns: the export's name, the format, the bound parameters, the outcome, the record count, a
checksum of the file and the error, plus the stored file and the audit columns. The ledger is
append-only and read-only, and serves each run's file for download.

**Normalisation** of a text field, applied in this order: `replace` (any number of pairs), `case:
upper | lower`, `allowed` with `onDisallowed: strip | space | fail` (default `fail`),
`collapseSpaces: true`, and `onLoss: fail` - the run is refused when the letters and digits of the
result differ from the value's, case aside. A refusal names the record and the field. The ledger of
an export that normalises carries one more read-only column: every value the normalisation changed,
the original beside what was written.

## Edge rules

- Every reference resolves at authoring time: an unknown field, hop, dimension, measure or parameter
  is an error, and a `fileName` token must name a declared parameter.
- `allowed` lists the characters themselves; `A-Z` is an inclusive range, a `-` first or last stands
  for itself, a backslash escapes the next character. It is never read as a pattern language.
- Authoring errors: a normalisation key on a number or a date field; a `case`, `onDisallowed` or
  `onLoss` value outside its vocabulary; an `allowed` with a backwards range or an unfinished escape;
  `onDisallowed` without `allowed`; `onLoss` without `replace` or `allowed` (case and spaces never
  change a letter).
- `onLoss` compares letters and digits only, so a `replace` that maps a letter (`"ß": "SS"`) counts
  as a loss under `onLoss: fail` - by design.
- The change trace is kept for a succeeded run; a failed run keeps its refusal in the error.
- `submit:` (signed submission) is reserved and refused until it is specified.
- Out of scope: hops across models, nested XML, validation against an XML schema.
- An export that normalises nothing has the same ledger as one written before normalisation existed.

## Prior art / workarounds

A hand-written class per file format per country, with a normalisation helper per file. Reference
implementation: https://github.com/eclipse-dirigible/dirigible/pull/7885 (the construct and its
ledger) and https://github.com/eclipse-dirigible/dirigible/pull/7999 (normalising a field's text).

## Specification text

**Anchor:** Declarative glue, a new `###` section after `imports - a file uploaded into rows`
(proposal 0079), before `rollups - denormalised parent totals`.

### exports - a file in an exacting format over a query

A VAT ledger for the tax authority, a payroll declaration, a ledger feed a bank consumes: a file
whose every byte position, character and checksum the receiver checks, filled from a query over the
model. The file is declared record by record and field by field, and every run is kept.

```yaml
exports:
  - name: SupplierDeclaration
    source: PurchaseInvoice
    format: fixedWidth
    encoding: windows-1251
    parameters: [{ field: issuedOn, op: between }]
    orderBy: [issuedOn, number]
    fileName: "DECL_{from}_{to}.txt"
    ledger: ExportRun
    header:
      - { literal: "HDR", width: 3 }
      - { param: from, width: 8, pattern: yyyyMMdd }
    line:
      - { from: supplier.vatNumber, width: 15, align: left, pad: " " }
      - { from: total, width: 12, align: right, pad: "0", scale: 2, decimalSeparator: "." }
    trailer:
      - { from: "count()", width: 6 }
      - { from: "sum(total)", width: 14, scale: 2 }
```

`source` is an entity or a plain [report](#reports). `format` is `fixedWidth`, `delimited` or `xml`;
`encoding` names the character set the file is written in. `parameters` are the run's inputs - over
an entity, filters `{ field, op }` on its fields; over a report, the report's own parameters - and
`orderBy` orders the rows. `fileName` may carry `{param}` tokens.

The file is three kinds of record: `header`, one `line` per row, and `trailer`. Each is a list of
fields, and a field reads exactly one of:

- `from:` - a field of the source, a one-hop `relation.field`, a report dimension or measure, or an
  aggregate over the lines, `sum(<field>)` or `count()`;
- `literal:` - a constant;
- `param:` - a bound parameter.

Its layout is `width`, `align`, `pad` and `truncate`; a number takes `scale` and
`decimalSeparator`, a date or a timestamp `pattern`; in an `xml` file, `name` is the element and
`attribute` makes the value an attribute.

`ledger:` names an entity the author declares **with no fields**: the export supplies them - the
export's name, the format, the bound parameters, the outcome, the record count, a checksum of the
file and the error - together with the stored file and the audit columns.

> **Normative.**
> Every reference of an export - a field, a hop, a dimension, a measure, a parameter, a `fileName`
> token - MUST resolve when the model is generated; one that does not MUST be reported as an
> authoring error. A field declaring more than one of `from:`, `literal:` and `param:`, or none,
> MUST be rejected.
>
> A conforming generator MUST serve each export as a download and as an entry point that a scheduled
> job or hand-written code can call, open to the roles granted the export's `read` permission.
>
> Nothing is written approximately. A value wider than its field that does not declare `truncate`,
> a character the declared encoding cannot represent, and a character XML does not allow MUST each
> fail the run; a failed run MUST be recorded on the ledger as FAILED and answered as unprocessable
> (422). A missing or unreadable parameter MUST be answered as a bad request (400) and
> MUST record nothing.
>
> Every other run MUST be recorded on the ledger with its parameters, outcome, record count,
> checksum and file. The ledger MUST be append-only and read-only to every user surface, and MUST
> serve the file of each run it records.
>
> `submit:` is reserved: a conforming generator MUST reject it until it is specified.

Hops across models, nested XML and validation against an XML schema are beyond this construct. A
model declaring no `exports` is unaffected.

#### Normalising a field's text

A regulator's format often accepts fewer characters than the master data holds, while the registered
value must stay as registered. A text field may therefore normalise what it writes, without touching
what is stored:

```yaml
line:
  - { from: supplier.name, width: 60,
      replace: { "ЕООД": "", "ООД": "" },   # applied first, any number of pairs
      case: upper,
      allowed: "А-Я0-9 -", onDisallowed: strip,
      collapseSpaces: true,
      onLoss: fail }
```

`allowed` lists the characters themselves: `A-Z` is an inclusive range, a `-` first or last stands
for itself, and a backslash escapes the next character; it is never read as a pattern language.
`onDisallowed` says what a character outside `allowed` becomes - `strip`, `space`, or `fail`, the
default. `onLoss: fail` refuses the run when the letters and digits of the result differ from those
of the value, case aside: a Latin name in a Cyrillic-only field writes nothing nobody typed.

> **Normative.**
> The normalisation MUST be applied in this order - `replace`, `case`, `allowed` with
> `onDisallowed`, `collapseSpaces`, `onLoss` - to the field's text before the encoding and XML
> checks. A disallowed character under `onDisallowed: fail`, or absent `onDisallowed`, and a
> loss under `onLoss: fail` MUST fail the run as above, the refusal naming the record and the field.
> `onLoss` compares letters and digits only; a `replace` that maps a letter therefore counts as a
> loss.
>
> The ledger of an export that normalises at least one field MUST carry one more read-only column,
> listing for a succeeded run every value the normalisation changed, the original beside what was
> written. An export that normalises nothing MUST have the same ledger as one that declares no
> normalisation key.
>
> Each of the following MUST be reported as an authoring error: a normalisation key on a number or a
> date field; a `case` other than `upper` / `lower`, an `onDisallowed` other than `strip` / `space` /
> `fail`, an `onLoss` other than `fail`; an `allowed` with a backwards range or an unfinished escape;
> `onDisallowed` without `allowed`; and `onLoss` without `replace` or `allowed`, since case and
> spaces never change a letter.

## DSL index

| Construct | What it gives you |
| --- | --- |
| [`exports`](#exports---a-file-in-an-exacting-format-over-a-query) | a fixed-width, delimited or XML file over an entity or a plain report, declared record by record and field by field, served as a download and a callable run; nothing is written approximately |
| [`exports.ledger`](#exports---a-file-in-an-exacting-format-over-a-query) | the entity, declared with no fields, on which every run is recorded append-only with its parameters, outcome, count, checksum and file |
| [`exports` field normalisation](#normalising-a-fields-text) | `replace` / `case` / `allowed` + `onDisallowed` / `collapseSpaces` / `onLoss`: a text field written in the characters the format accepts, refused rather than written with letters lost, every change kept on the ledger |
