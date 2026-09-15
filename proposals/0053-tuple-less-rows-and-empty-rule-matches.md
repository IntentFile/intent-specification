# A row outside every tuple is not guarded, and a rule match is a literal that says something

- **Status:** released in [1.8](../versions/1.8.md)
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7180
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7253

## The problem

Two constructs are specified in terms of a lookup by value, and neither says what the lookup does
when the value is absent. Both readings exist in the wild, and each is silent in its own way.

**A guard over an aggregate.** `aggregates` is normative about the null case: "A source row with any
grouping key unset belongs to no tuple and MUST be ignored." `checks: kind: guard` recomputes the
same total for the incoming row's key-tuple and is normative about the recompute, but says nothing
about a row that has no tuple. So a movement authored without its warehouse - one of the two
grouping keys - is weighed against a total, and the only question is which one. Read as "no rows
match", the guard compares the incoming amount against zero and rejects a withdrawal that breaches
nothing. Read as "the rows whose key is also unset", it compares it against a pool of rows the
aggregate itself ignores, for which no target row is ever materialised. The two readings differ on
whether a write is accepted, and an implementation can move between them by changing how it renders
a comparison with no value - which is exactly what happened.

**A posting's determination rule.** `rule: { entity: PostingRule, match: { documentType: "..." } }`
selects the rule row a posting derives its accounts from, and the specification calls the value a
literal. It does not say the literal has to be there. An empty one - the key authored with nothing
after it, or with an empty string - is a lookup for the empty value: it matches no rule row, so the
posting takes its documented "no rule row" path and skips the source document to the unposted
worklist. Every source document. The model is well-formed, the generation succeeds, the deployment
is healthy, and the only symptom is documents that are never posted.

## The proposed shape

No new key in either case. Both are the null case of an existing one.

```yaml
- name: StockMovement
  relations:
    - { name: Product, kind: manyToOne, to: Product }
    - { name: Warehouse, kind: manyToOne, to: Warehouse }   # optional - may be unset
  checks:
    - kind: guard
      aggregate: onHand        # by: [Product, Warehouse]
      minimum: 0
      message: "Insufficient stock"

postings:
  - name: goodsIssuePosting
    event: { onTransition: GoodsIssue, when: "Status == Issued" }
    creates: JournalEntry
    backReference: GoodsIssue
    rule: { entity: PostingRule, match: { documentType: "Goods Issue" } }   # a literal, and not an empty one
    items:
      - { Account: rule(costOfSalesAccount), debit: "CostValue" }
      - { Account: rule(inventoryAccount),   credit: "CostValue" }
```

## Expected behaviour

**The guard.** A record with any of the aggregate's grouping keys unset belongs to no tuple. It
contributes to no total and can therefore breach no minimum, so a conforming generator MUST NOT
apply the guard to it: the write proceeds, no violation is reported, a `marker` reads as holding and
a `setStatus` is not written. This is the guard's half of the rule `aggregates` already states, and
it keeps the guard and the aggregate two computations of the same total rather than of two different
ones.

**The rule match.** `match` is a single `column: literal` selector whose literal MUST be present and
non-blank. An absent or blank value MUST be reported as an authoring error when the intent is read.
It cannot be treated as a match on the empty value: that is indistinguishable from a rule table that
has not been filled in yet, and it silences the whole posting rather than one document.

## Edge rules

- The guard's non-application is not an outcome. It is not `block`, `task` or `reject`: nothing is
  refused, nothing is marked, nothing is filed rejected. An implementation SHOULD record that it
  skipped, naming the guard, so a record that is never guarded can be traced to the key it lacks.
- A record acquiring its missing key is an ordinary update, and the guard applies to it from then on
  - as does the aggregate.
- The guard's skip says nothing about requiredness. A grouping key that must always be there is
  declared `required`, which is a separate statement and the better one where it is true.
- The rule-match rule is about the authored literal only. A rule ROW whose match column is empty
  remains an ordinary row of the rule table, and it is matched by no non-blank literal.

## Prior art / workarounds

Both are worked around by declaring more than the model means. A grouping key is marked `required` so
the tuple-less case cannot arise - correct where the value really is mandatory, and a fabricated
constraint where it is not. A rule match is verified by publishing the application and watching
whether documents actually get posted, which is the only signal an empty literal produces.

## Specification text

Two pieces of text, each at its own anchor.

**Anchor:** Entities & fields > checks — declarative validations > kind: guard — a precondition
over an aggregate, appended after the **Normative** paragraph and before "`outcome: task` sets a
flag; it does not itself create or route to a task."

A record with any of the aggregate's grouping keys unset belongs to no tuple — the half of the
rule [`aggregates`](#aggregates--keyed-cross-entity-totals) already states. It contributes to no
total and can breach no minimum, so the guard is not applied to it. Its non-application is not an
outcome: nothing is refused, nothing is marked, nothing is filed rejected. A record acquiring its
missing key is an ordinary update, and the guard applies to it from then on, as does the
aggregate. The skip says nothing about requiredness: a grouping key that must always be there is
declared `required`, a separate and better statement where it is true.

> **Normative.** A conforming generator MUST NOT apply a guard to a record with any of the
> aggregate's grouping keys unset: the write proceeds, no violation is reported, a `marker` takes
> the value a holding guard gives it, and a `setStatus` is not written. It MUST NOT compare the
> record against the total of a zero-row tuple, nor against the rows whose key is also unset. A
> conforming generator SHOULD record that it skipped, naming the guard, so a record that is never
> guarded can be traced to the key it lacks.

<!-- editor: the proposal says a `marker` "reads as holding" for a tuple-less record; whether the
     generator writes `true` or merely leaves the field is not stated. The text above follows the
     outcome table ("`true` whenever the guard holds") and requires the holding value. -->

**Anchor:** Declarative glue > postings — source document to ledger, appended after the first
example (the one introducing `rule: { entity: PostingRule, match: { documentType: "Sales Invoice" } }`)
and before the paragraph "The trigger is `onTransition` …".

`rule.match` is a single `column: literal` selector: the literal selects the rule row the posting
derives its accounts from. The literal must be present and non-blank. An empty one is not a match
on the empty value — that is indistinguishable from a rule table not yet filled in, and it would
silence the whole posting rather than one document — so it is an authoring error, reported when
the intent is read. The rule is about the authored literal only: a rule ROW whose match column is
empty remains an ordinary row of the rule table, matched by no non-blank literal. A rule row that
is absent at run time keeps its documented path — the source document is skipped to the unposted
worklist.

> **Normative.** `match` MUST be a single `column: literal` selector whose literal is present and
> non-blank. An absent or blank literal MUST be reported as an authoring error when the intent is
> read; a conforming generator MUST NOT treat it as a lookup for the empty value.

## DSL index

No construct is added; both texts are the null case of an existing key. The
`checks: kind: guard` and `postings` rows are unchanged.
