# imports - an uploaded CSV, MT940 or camt.053 file is read into entity rows, whole or not at all

- **Status:** released in [1.12](../versions/1.12.md)
- **Issue:** https://github.com/IntentFile/intent-specification/issues/140, https://github.com/eclipse-dirigible/dirigible/issues/7856, https://github.com/eclipse-dirigible/dirigible/issues/8190
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7884, https://github.com/eclipse-dirigible/dirigible/pull/8325

## The problem

"Read this file into rows" - a bank statement, a supplier's price list - has no declarative
construct in 1.11. [Seeds](../versions/1.11.md#seeds) are reference data loaded at publish time, and
an [`inbound`](../versions/1.11.md#inbound--arrivals-from-outside) arrival takes "a JSON record shaped
like the entity", not a file with an encoding, a locale, a header line and a rule for what a
repeated upload means. So every statement import was a hand-written parser per format, and each one
decided on its own what happens to a file that is half readable or uploaded twice.

The obvious name is taken. In 1.11, `imports:` is an **entity** key that "injects import lines into
the generated data-access layer (pairs with calculated actions)". The construct below is therefore a
**top-level** collection, `imports[]`, and an entry names the entity it fills with `into:`, never
with an entity-level key, so the two cannot be confused.

## The proposed shape

```yaml
imports:
  - name: bankStatement
    into: StatementLine
    format: mt940                        # csv | mt940 | camt053
    map:
      statementId: statementId           # the vocabulary the statement readers share
      lineNumber:  lineNumber
      valueDate:   valueDate
      amount:      amount
      reference:   reference
      fileName:    _file                 # _file / _line: facts of the upload itself
    duplicateKey: [statementId, lineNumber]

  - name: priceList
    into: PriceListLine
    format: csv
    csv: { separator: ";", decimal: ",", grouping: ".", dateFormat: dd.MM.yyyy,
           header: true, skipLines: 0, encoding: windows-1251 }
    map: { sku: SKU, price: Price }      # a field of PriceListLine <- a column of the file
```

A model without `imports` generates exactly as in 1.11.

## Expected behaviour

- The generated application offers an **upload** per import, open to the roles allowed to write the
  `into` entity, and an entry point that a scheduled job or hand-written code can call with a file
  name and the file's bytes.
- The file is **decoded strictly**: a byte sequence the declared encoding cannot represent refuses
  the file; it is never stored garbled. Values are **converted strictly** under the declared locale
  (separator, decimal and grouping marks, date and timestamp patterns).
- `map` fills a field of `into` from a CSV column, from the statement vocabulary the `mt940` and
  `camt053` readers share, or from `_line` (the record's position in the file) and `_file` (the
  uploaded file's name). An MT940 file may hold several statements.
- The **whole file is stored as one unit**: a value that cannot be read answers **400**, a
  `duplicateKey` the target already holds answers **409**, and in either case **nothing** of the
  file is stored.
- Every run is recorded on the target's **import ledger** - file name, outcome, row count, error.
- Hand-written code may react to an import after it is committed, over the rows it stored.

## Edge rules

- `duplicateKey` lists mapped fields of `into`. It is enforced by a **data-store constraint** on the
  target, derived from the key - one constraint per distinct key, shared with a hand-authored
  [`unique`](../versions/1.11.md#unique--a-business-key-over-more-than-one-field) over the same fields
  and with another import into the same entity declaring the same key. Two simultaneous uploads of
  one file therefore cannot both pass: at most one commits, and the other is answered 409 exactly as
  a later upload of the same file would be.
- A camt.053 document that declares a document type is refused, so a file cannot make the reader
  resolve external entities.
- A relation of `into` is not filled from the file: there is no lookup.
- Out of scope: signed submission, a ledger surface beyond its list, XML schema validation.

## Prior art / workarounds

A hand-written parser and controller per file format, each with its own idea of atomicity and of a
repeated upload; seed imports for anything that could wait for a publish. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/7884 (the construct) and
https://github.com/eclipse-dirigible/dirigible/pull/8325 (the duplicate key as a data-store
constraint, closing the race between two simultaneous uploads).

## Specification text

**Anchor:** Declarative glue, a new `###` section after `outbound - departures on a queue or topic`
(so that the arrival and the departure stay together), before `rollups - denormalised parent
totals`.

### imports - a file uploaded into rows

A bank statement, a supplier's price list, a counterparty's remittance file: someone hands the
application a **file**, and its records become rows of an entity. That is neither a [seed](#seeds),
which is reference data loaded at publish, nor an [arrival](#inbound--arrivals-from-outside), which
carries a JSON record; a file has an encoding, a locale, a header line, and an answer owed to the
person who uploaded it twice.

```yaml
imports:
  - name: bankStatement
    into: StatementLine
    format: mt940
    map:
      statementId: statementId
      lineNumber:  lineNumber
      valueDate:   valueDate
      amount:      amount
      reference:   reference
      fileName:    _file
    duplicateKey: [statementId, lineNumber]

  - name: priceList
    into: PriceListLine
    format: csv
    csv: { separator: ";", decimal: ",", grouping: ".", dateFormat: dd.MM.yyyy,
           header: true, skipLines: 0, encoding: windows-1251 }
    map: { sku: SKU, price: Price }
```

`imports` is a **top-level** collection. It is unrelated to the entity-level `imports:` key, which
injects import lines into the generated data-access layer; an entry names the entity it fills with
`into:`, and an entity's own `imports:` keeps its meaning.

`format` is `csv`, `mt940` or `camt053`. A `csv` import describes its file under `csv:` -
`separator`, `decimal`, `grouping`, `dateFormat`, `timestampFormat`, `header`, `skipLines` and
`encoding`. `map` fills a field of `into` from one source per key: a CSV column, a term of the
statement vocabulary the `mt940` and `camt053` readers share (`statementId`, `lineNumber`,
`valueDate`, `amount`, `reference`, ...), `_line` - the record's position in the file - or `_file`,
the uploaded file's name. `duplicateKey` lists mapped fields of `into` that identify a row across
uploads.

> **Normative.**
> A conforming generator MUST offer, for each import, an upload open to the roles allowed to write
> the `into` entity, and an entry point a scheduled job or hand-written code can call with a file
> name and its content.
>
> The file MUST be decoded strictly in its declared encoding: a byte sequence the encoding cannot
> represent refuses the file, and nothing is ever stored with a substituted character. Every value
> MUST be converted strictly under the declared locale - the separator, the decimal and grouping
> marks, the date and timestamp patterns. A camt.053 document declaring a document type MUST be
> refused.
>
> The rows of one file MUST be stored as **one unit**: either every row is stored or none is. A
> value that cannot be read MUST be answered as a bad request (400), and a row whose `duplicateKey`
> the target already holds as a conflict (409); in either case nothing of the file is stored.
>
> `duplicateKey` MUST be enforced by the data store, as a
> [`unique`](#unique--a-business-key-over-more-than-one-field) key is, so that two simultaneous
> uploads of one file cannot both commit; the one that does not commit MUST be answered as a
> conflict, exactly as a later upload of the same file would be. A key over the same fields as a
> declared `unique`, or as another import into the same entity, is one constraint, not two.
>
> Every run MUST be recorded on the target's import ledger with the file name, the outcome, the row
> count and the error, if any. Hand-written code MAY react to an import once it is committed, over
> the rows it stored; a refused file reaches no such reaction.
>
> `into` MUST name an entity of the model; each key of `map` MUST name a field of `into`; each
> element of `duplicateKey` MUST name a field `map` fills; `format` MUST be one of the three values.
> Each violation MUST be reported as an authoring error.

An import fills fields only: a relation of `into` is not resolved from a value in the file. Signed
submission, a ledger surface beyond its list, and validation against an XML schema are beyond this
construct. A model declaring no `imports` is unaffected.

## DSL index

| Construct | What it gives you |
| --- | --- |
| [`imports`](#imports---a-file-uploaded-into-rows) | a top-level collection: an uploaded CSV / MT940 / camt.053 file read into rows of the `into` entity, strictly decoded, stored whole or not at all, every run on an import ledger (not the entity-level `imports:` key) |
| [`imports.duplicateKey`](#imports---a-file-uploaded-into-rows) | the mapped fields that identify a row across uploads, enforced by the data store: a repeated key refuses the whole file with a conflict, simultaneous uploads included |
