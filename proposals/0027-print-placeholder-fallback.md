# Alternative operands in a print placeholder

- **Status:** draft
- **Issue:**
- **Reference implementation:** https://github.com/eclipse-dirigible/dirigible/issues/6900

## Why

A print-template placeholder that resolves to an empty value renders blank, and there is no way to
say "and if that one is empty, print this one instead".

That is workable for a template whose data is filled once - a tenant's own profile - and it becomes a
data-quality trap the moment a template reads a field that is *optional by design* across hundreds of
records. The shape recurs everywhere in business data: a partner carries a locally registered name
beside its canonical one, a short trading name beside the legal name, a local-language address beside
the transliterated one. Some records have both, some only one.

The template author then has to pick a single field, and every record that happens to have the *other*
one filled prints a hole - in a legal document, at the point where the counterparty's name belongs.
Neither the author nor the recipient of the document sees anything wrong until someone reads it.

The workarounds are all worse than a fallback:

- **normalise the data** - copy the canonical value into the optional field for every record that
  lacks one, which destroys the distinction the two fields exist to record;
- **two templates**, selected per record, which no per-record selection mechanism exists for;
- **a hand-maintained combined field** on the entity, filled by the writers that remember to.

## What this adds

Alternative operands inside one placeholder, first non-blank wins, left to right:

```text
<field label="Customer">{{document.Customer.NameLocal|document.Customer.Name}}</field>
```

That is the whole addition. Every operand is an ordinary path with the ordinary path rules; there
are no literals, no expressions, and no other new syntax.

## The normative half

**"Blank" is null, missing, or whitespace-only.** All three are the same thing to a reader of the
printed document - a hole - so all three fall through.

**Every operand resolves exactly as a single path does**, including inside a row scope: the row
first, then the enclosing document. A fallback is not a second, weaker resolution mode.

**The last operand is rendered whatever it holds.** That is what makes the addition safe rather than
merely convenient: a single path is the one-operand case of the rule, so its behaviour is unchanged,
and a placeholder whose operands are all blank renders empty exactly as an unresolved single path
already does. An existing template is byte-identical until it adopts the syntax.

Nothing is added to the *parser*: the placeholder body is already opaque to it. The change is in the
layer that merges data into a placeholder.

## Notes

Deliberately **not** part of this proposal:

- **Literal operands** (`{{a|b|"n/a"}}`). A printed placeholder for a value nobody filled is a
  business decision about the document, and the template already has `if` to make it.
- **Any operator or expression.** The reason this construct is safe to add is that it is a list of
  paths and a rule for choosing among them.
- **Applying the same syntax to the `source` / `filter` / `match` attributes.** Those select rows,
  not text, and the failure mode this fixes is a hole in rendered text.

## Specification text

The prose below is what a release folds into the next version document, at the anchors given.

**Anchor:** Printable documents (a new subsection, before "Row filtering")

#### Alternative operands in a placeholder

A placeholder may list several paths separated by `|`; the first one resolving to a non-blank value
is rendered, left to right:

```text
<field label="Customer">{{document.Customer.NameLocal|document.Customer.Name}}</field>
```

This is the fallback an optional twin field needs. A record carrying a locally registered name beside
its canonical one has one filled or both, and a template that had to name a single field printed a
hole in the document for every record with the other one.

> **Normative.**
> Every operand MUST resolve by the same rules as a single path, including row-scope resolution
> inside a table or a row-expanding node.
> A value counts as blank when it is null, missing, or whitespace-only.
> The LAST operand MUST be rendered whatever it resolves to, so that a single path is the
> one-operand case of the rule and a placeholder whose operands are all blank renders empty - exactly
> as an unresolved single path does.
> Any number of operands is allowed. No other syntax is introduced: an operand is a path, never a
> literal or an expression.
> A template using this syntax rendered by an implementation that predates it MUST NOT fail to parse.

**Anchor:** Appendix A: DSL index

| [`{{a\|b}}`](#alternative-operands-in-a-placeholder) | a print placeholder's alternative paths - first non-blank wins |
