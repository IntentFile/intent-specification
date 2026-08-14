# Print-template row filtering (`filter` / `match`)

- **Status:** draft
- **Issue:** <!-- link the discussion issue, if any -->
- **Proposed for:** the printable-documents template language (versions/1.1.md "Printable documents")

## The problem

A print template's `table` renders **every** element of the collection it is bound to, and `if`
tests only the truthiness of a single path. One collection therefore cannot render into several
purpose-grouped tables - yet that is the shape of the most ordinary financial documents:

- a **payslip** prints earnings in one column and deductions in the other, both from the same
  fiche-line items;
- a **journal entry** prints its debit side next to its credit side, both from the same line rows;
- an **invoice VAT summary** groups the same items per rate.

The line items of such a document carry a discriminator field (a line kind, a side, a rate group),
but the template language has no way to say "this table shows only the rows whose kind is X".
Today the only options are pre-splitting the data feed (the generated feeders deliberately feed
the entity's items as ONE collection) or abandoning the grouped layout.

## The proposed shape

Two attributes, valid on `table` (and the row-expanding `for`), plus `match` on `if`:

```text
<row gap="16">
    <stack>
        <text style="subtitle">Earnings</text>
        <table source="items" filter="Kind" match="BASE | ENTRY">
            <column width="3*" label="Earning">{{Name}}</column>
            <column width="*" align="right" label="Amount">{{Amount}}</column>
        </table>
    </stack>
    <stack>
        <text style="subtitle">Deductions</text>
        <table source="items" filter="Kind" match="CONTRIBUTION | TAX">
            <column width="3*" label="Deduction">{{Name}}</column>
            <column width="*" align="right" label="Amount">{{Amount}}</column>
        </table>
    </stack>
</row>

<if source="status" match="POSTED | SENT">
    <text>Final document</text>
</if>
```

- `filter="<path>"` on a `table`/`for` names a path resolved **in each row's own scope**.
  Without `match`, a row is kept when the resolved value is truthy.
- `match="A | B"` lists accepted literal values, `|`-separated, surrounding whitespace trimmed.
  A row is kept when the resolved value's string form equals one of the literals.
- `match` on an `if` compares the node's resolved `source` against the listed literals instead
  of testing truthiness.

## Expected behaviour

- A filtered `table` renders its column definitions unchanged and one row per KEPT element, in
  source order; elements failing the filter are skipped entirely (no empty rows).
- A filtered `for` expands its children only for the kept elements.
- An unresolved / null filter value never matches a `match` list, and is falsy without one.
- Value comparison is by the value's plain string form - numbers compare by their canonical
  rendering, booleans as `true`/`false`. No coercion beyond that, no operators, no expressions:
  the language stays a declarative value match.
- A template written with `filter`/`match` and rendered by a generator that predates them MUST
  degrade to rendering all rows (unknown attributes are ignored), never fail to parse.

## Edge rules

- `filter` absent: the table/for renders every element (today's behaviour, unchanged).
- `match` present without `filter` on a `table`/`for`: no effect (there is nothing to compare);
  conforming implementations should ignore it.
- Empty `match` (blank string) is treated as absent: plain truthiness of the filtered path.
- A literal containing `|` is not expressible; the discriminators this feature targets are
  code-like line kinds, which never contain pipes.
- `match` on `if` without a `source` keeps the children hidden (a null never matches).

## Prior art / workarounds

Hand-maintained pre-split collections in a custom data feeder, or a single mixed table with a
kind column - both give up the standard two-column financial layout. Report-side grouping does
not help: the print template renders a document's OWN items, not a report. The reference
implementation ships the behaviour in its template data-binding layer with parser-level
compatibility (the attributes parse as plain strings like every other attribute).
