# A print placeholder may carry a format pattern

- **Status:** released in [1.9](../versions/1.9.md)
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/6982
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/6983

## The problem

A print template binds values with placeholders - `{{document.Total}}`, `{{Price}}` inside the items
table - and the author has no say in how a bound value is written out. The rendering layer decides
from the value's shape: a fractional number is written in the implementation's money pattern, a whole
number is written bare, a date is written in the ISO form the data arrives in.

That is fine until the shape of a value stops matching what it *means*. A price of 5390.00 is money,
but by the time it reaches the renderer its scale marker is gone - a whole figure serialised on the way
through the browser is `5390`, indistinguishable from an id or a year - and it prints bare next to a
formatted totals block:

```
QTY    PRICE       TOTAL
  2    2 863.24    5 726.48
  1    5390        5 390.00
```

A real invoice shows this as a mixed column: amounts with cents formatted, round amounts bare.
Dates have the sibling problem: `2026-08-29` cannot be written as `29.08.2026`, the convention the
document's readers expect, and `2026-08-29T10:41:22` cannot be cut down to `10:41`.

Nothing the template author writes fixes either. The fix is not to blanket-format every whole number
as money - an id, a year, a quantity or a line number printed through the same placeholder would come
out as `7.00` - so the renderer cannot know; only the author does, and says so per placeholder.

## The proposed shape

An optional format pattern on a placeholder operand, after the operand's first colon:

```text
<column width="*" align="right" label="PRICE">{{Price:#,##0.00}}</column>
<text align="right">Date: {{document.Date:dd.MM.yyyy}}</text>
<text>{{At:HH:mm}}</text>                                   <!-- the FIRST colon splits -->
<text>{{A:0.00|B:#,##0.00}}</text>                          <!-- per operand, composed with | -->
<text>{{document.Customer.NameLocal|document.Customer.Name}}</text>   <!-- unchanged -->
```

The construct is a suffix on an operand, not a new operand kind: the operand is still an ordinary
path, resolved by the ordinary rules, and the pattern says how the value it resolves to is written.
It mirrors the `{field:pattern}` token that [`fileName`](../versions/1.6.md#naming-the-rendered-file--filename)
patterns already carry for dates.

## Expected behaviour

A conforming generator's rendering layer, when a placeholder operand carries a pattern:

- **A number** - of any kind, whole numbers included, which is the reason the construct exists -
  is written through the pattern as a **decimal pattern**: `#,##0.00` groups thousands and writes
  exactly two fraction digits, `0.00` writes two fraction digits without grouping, `#` and `0` carry
  their usual optional / mandatory digit meaning. `{{Price:#,##0.00}}` writes `5390` as `5 390.00`.
- **A temporal value** - a date, a date-time, a date-time with an offset - or **text in the ISO form
  a feeder emits for one** (`2026-08-29`, `2026-08-29T10:41:22`, `2026-08-29T10:41:22+02:00`) is
  written through the pattern as a **date-time pattern**, with the usual letters (`yyyy`, `MM`, `dd`,
  `HH`, `mm`, `ss`). `{{document.Date:dd.MM.yyyy}}` writes `2026-08-29` as `29.08.2026`.
- **Any other value, and any pattern the value cannot satisfy**, is written exactly as it would be
  with no pattern. A text that is not an ISO temporal, a boolean, a related record's display label,
  a malformed pattern, a decimal pattern over a date, a time pattern (`HH:mm`) over a date-only
  value: each renders the default way. A printout never shows an error, a raw placeholder, or
  anything the value did not contain - the print template's standing leniency contract.
- **An operand with no pattern renders as before.** No template in existence carries a colon inside
  a placeholder body, so every existing template - generated or hand-adapted - renders byte-identical.

Composition with the alternatives of `{{a|b}}` is per operand: each operand carries its own pattern
or none, blankness is judged on what the operand *renders*, and the last operand renders whatever it
holds. `{{A:0.00|B:#,##0.00}}` writes `A` to two decimals when it is filled, otherwise `B` grouped.

## Edge rules

- **The first colon splits.** Everything before it is the path, everything after it - surrounding
  whitespace trimmed - is the pattern. So a time pattern keeps its own colons: `{{At:HH:mm}}` is the
  path `At` with the pattern `HH:mm`. A path never contains a colon, so nothing is lost.
- **An empty pattern is no pattern.** `{{Price:}}` renders as `{{Price}}`.
- **The pattern applies to the resolved value, after relation resolution.** A placeholder that names a
  related record renders that record's display label; a pattern on it applies to the label - which is
  text, so a decimal or date pattern does not apply and the label renders as before.
- **Type dispatch is by the value, not by the pattern.** A number is always tried as a decimal
  pattern, a temporal always as a date-time pattern. A number is never parsed as a date; ISO text is
  the one shape that crosses from text into temporal, because it is how a feeder hands a date over.
- **Fallback is lenient, deliberately, and asymmetric with `fileName`.** A `fileName` pattern is part
  of the model and is validated at authoring time - a date format on a non-date field is rejected. A
  print template's placeholder body is opaque to the parser and rendered against data whose shape is
  only known at render time, so an inapplicable pattern falls back instead. The two constructs share
  the pattern language, not the failure mode.
- **Blankness under `|` is judged on the rendered text.** A null operand renders empty whether or not
  it carries a pattern and falls through; a formatted number is never blank, so a filled numeric
  operand always wins.
- **Symbols are the implementation's, not the render language's - in this version.** The pattern
  fixes *where* grouping and the decimal mark fall; the characters used are an implementation
  constant, the same for every render of every template regardless of the render language chosen. The
  reference implementation writes a locale-neutral form: a **space** as the grouping separator and a
  **dot** as the decimal mark, and date-time pattern letters that produce words (`MMMM`, `EEEE`) in
  their locale-neutral (English) form. Whether the render language should govern these symbols is
  left open - see Notes - and until it is decided, a conforming implementation MUST NOT vary them per
  render.
- **Contradiction with the current text.** The 1.6 Normative block under "Alternative operands in a
  placeholder" says *"No other syntax is introduced: an operand is a path, never a literal or an
  expression."* The shipped behaviour introduces exactly one piece of syntax on an operand, so the
  Specification text below replaces that sentence: the operand remains a path, and may now be
  followed by a format pattern; it is still never a literal or an expression.

## Prior art / workarounds

Before the construct shipped, the only remedies were on the data side: make sure every money value
reaches the renderer with a fractional part (impossible to guarantee across a browser round trip that
strips a whole number's scale), or add a text twin of each formatted field to the entity and keep it
in step - the kind of hand-maintained duplicate the format exists to remove. Dates had no remedy at
all: a template author who wanted `29.08.2026` could not have it.

The reference implementation resolved the pattern in the layer that merges data into a placeholder,
leaving the template parser untouched - the placeholder body was already opaque to it - and reuses the
symbol set the generated forms already print money with, so a formatted whole number and a
default-formatted fractional one are indistinguishable in the same column.

## Specification text

The prose below is what a release folds into the next version document, at the anchors given.

**Anchor:** Printable documents > Alternative operands in a placeholder (a new subsection
immediately after it, before "Row filtering — `filter` / `match`")

#### A format pattern on a placeholder operand

An operand may carry a format pattern after its first colon, saying how the value it resolves to is
written:

```text
<column width="*" align="right" label="PRICE">{{Price:#,##0.00}}</column>
<text align="right">Date: {{document.Date:dd.MM.yyyy}}</text>
<text>{{At:HH:mm}}</text>
<text>{{A:0.00|B:#,##0.00}}</text>
```

The renderer cannot know that a whole figure is money whose scale was lost on the way in, nor which
date convention a document's readers expect; the author knows, and says so per placeholder. A number
formats through a decimal pattern - grouping and fraction digits - and a temporal value, or the ISO
date or date-time text a feeder emits for one, through a date-time pattern. This is the pattern
language `fileName`'s `{field:pattern}` token already uses for dates.

> **Normative.**
> The FIRST colon in an operand separates the path from the pattern; the path is everything before
> it, the pattern everything after it with surrounding whitespace trimmed. A path never contains a
> colon, so `{{At:HH:mm}}` is the path `At` with the pattern `HH:mm`. An empty pattern is no pattern.
> A pattern applies to the value the operand resolves to, after the ordinary path and row-scope
> resolution and after a related record has been reduced to its display label.
> A value that is a number - whole numbers included - MUST be written through the pattern as a
> decimal pattern. A value that is a date, a date-time or an offset date-time, or text in the ISO
> date / date-time form, MUST be written through the pattern as a date-time pattern.
> A value of any other shape, and a value that cannot satisfy the pattern it is given - a malformed
> pattern, a pattern of the wrong kind, a pattern asking for a field the value does not carry - MUST
> render exactly as it would with no pattern. A pattern MUST NOT make a render fail, and MUST NOT
> surface a raw placeholder or an error in the rendered document.
> An operand with no pattern MUST render exactly as before this construct; an existing template
> renders unchanged.
> The pattern is per operand and composes with alternative operands: each operand carries its own
> pattern or none, blankness is judged on the text the operand renders, and the last operand renders
> whatever it holds.
> The pattern fixes the positions of grouping and of the decimal mark; the SYMBOLS written for them,
> and the words a date-time pattern produces for month and day names, are an implementation constant
> that MUST be documented and MUST NOT vary between renders or with the render language. The
> reference form is locale-neutral: a space as the grouping separator, a dot as the decimal mark.
> A template using this syntax rendered by an implementation that predates it MUST NOT fail to parse.

In the Normative block of "Alternative operands in a placeholder", the sentence

> Any number of operands is allowed. No other syntax is introduced: an operand is a path, never a
> literal or an expression.

is replaced by

> Any number of operands is allowed. An operand is a path, optionally followed by a format pattern
> after its first colon (see [a format pattern on a placeholder operand](#a-format-pattern-on-a-placeholder-operand));
> it is never a literal or an expression.

## DSL index

| Construct | What it does |
| --- | --- |
| [`{{a:pattern}}`](#a-format-pattern-on-a-placeholder-operand) | a print placeholder operand's format pattern - decimal for numbers, date-time for temporals; inapplicable falls back |

## Notes

Deliberately **not** part of this proposal:

- **Render-language-sensitive symbols.** Proposal 0029 made the render language govern the language
  a document's *data* is read in. Whether it should also govern the grouping separator, the decimal
  mark and the month and day names a pattern produces is a real question - a Bulgarian invoice
  writes `5 390,00`, a US one `5,390.00` - and a different one: it needs a symbol set per language
  and a rule for a language the implementation has no symbols for. This proposal fixes the shipped
  behaviour, a single implementation-constant symbol set, and leaves that door marked.
- **Literal operands or expressions.** The pattern is a rendering instruction on a path's value, not
  a value of its own; the reason the alternatives construct is safe to extend stays intact.
- **A default pattern per field type or per entity field.** A model-level `format:` on a field would
  reach every surface, not only print; it is a separate proposal if wanted.
