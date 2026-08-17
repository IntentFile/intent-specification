# A report column of a multilingual entity is read in the caller's language


- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/6544
- **Discussion:** https://github.com/IntentFile/intent-specification/pull/25

## What this clarifies

The Multilingual data chapter already says that *every read* of a translatable property is served in the caller's requested language. It never said whether a **report** column counts as such a read — and a report is not an entity read at all, it is an aggregation over stored values, so an implementation could reasonably go either way.

In practice implementations went the other way, and the result is visibly wrong to a user: a report grouping by a multilingual nomenclature showed the stored term (`DRAFT`) right beside a list page showing the translated one, for the same record.

## What it says

Two sentences of prose plus a Normative paragraph in **Multilingual data → Data**, and the Appendix A row:

- A report column bound to a translatable property is a read of that property, so it is served in the caller's requested language — whether the report is rooted at the multilingual entity or reaches it through a relation.
- **The boundary that makes this safe:** what a report *matches* is unaffected. A report's `filter:`, its `scope:` and any condition applied to it are evaluated against the stored, untranslated values, so translating content can never change which rows a report returns, only how they read.
- A property with no translation for the requested language, and a caller who requested none, both read the stored value.

No new construct and no new keyword — this constrains what a conforming generator must do with the existing `multilingual:` attribute.

## Reference implementation

Proven out in the reference implementation by [eclipse-dirigible/dirigible#6750](https://github.com/eclipse-dirigible/dirigible/pull/6750) (issue [#6544](https://github.com/eclipse-dirigible/dirigible/issues/6544)): the translated column is emitted as a `COALESCE` over a `LEFT JOIN` to the entity's translation table keyed on a bound language parameter, so the fallback and the filter boundary both fall out of the SQL rather than being enforced separately. Its integration test asserts the same published report returning the translated term under one request language and the stored one under another.

## Specification text

The prose below is what a release folds into the next version document, at the anchors
given. It was written against `versions/1.2.md` and is carried here unchanged.

**Anchor:** Data, seeds & naming > Multilingual data > Data

A [report](#reports) reads the same data, so it reads it in the same language: a column bound to a translatable property is shown translated, whether the report is rooted at the multilingual entity or reaches it through a relation. A report grouping by a multilingual nomenclature therefore shows the same term as the pages beside it — before this was stated, a status column could read `DRAFT` next to a list reading the translated word, from the same record.

> **Normative.**
> Every read of a translatable property of a `multilingual: true` entity is served in the caller's requested language, and a report column bound to such a property is such a read. What a report **matches** is unaffected: a report's `filter:`, its [`scope`](#lifecycle-scope) and any condition applied to it are evaluated against the stored, untranslated values — so translating content can never change which rows a report returns, only how they read. A property with no translation for the requested language, and a caller who requested none, both read the stored value.

**Anchor:** Appendix A: DSL index

| [`multilingual` / `languages`](#multilingual-data) | translation tables + read-time translation overlay, on entity reads and report columns alike |
