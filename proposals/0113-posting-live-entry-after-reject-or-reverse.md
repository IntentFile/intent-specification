# The post an amended source rewrites is its live one: a rejected or reversed entry is retired and a new draft is created

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/163, https://github.com/eclipse-dirigible/dirigible/issues/7959
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8438
- **Amends:** [0032 - an amended source rewrites its posting](0032-amended-source-rewrites-its-posting.md), [0075 - an `onCreate` posting follows its source](0075-on-create-posting-follows-its-source.md)

## The problem

0032 and 0075 compare the source with "the post the `backReference` finds". The back-reference is
at most once per source, but over a document's life it can find more than one created document,
and 1.11 does not say which one is "the" post. Two ordinary paths leave the first one found dead:

- The accountant **rejects** the draft entry: it stands in a cancelled status.
- The entry is **posted and then reversed** by a posted [`reversals`](0083-reversals.md) twin - and
  the twin, a copy of the entry, carries the same back-reference.

Neither is in the status the posting's create gave it, so on the next edit of the source 1.11 reads
"the created document has left that status": the divergence is reported, the correction is left to
`reverses`, and the corrected source stays off the ledger, although nothing live represents it.

## The proposed shape

No new key. The rule reads two declarations the file already makes: the
[`stage:`](#stage--what-a-status-means-to-the-lifecycle) classification of the target's status
seeds, and a `reversals` entry on the target.

```yaml
seeds:
  - name: journal-entry-statuses
    entity: JournalEntryStatus
    rows:
      - { id: 1, name: DRAFT,     stage: draft }
      - { id: 2, name: POSTED,    stage: live }
      - { id: 3, name: CANCELLED, stage: cancelled }   # a rejected draft is retired

reversals:
  - { name: reverseEntry, forEntity: JournalEntry, when: "Status == POSTED",
      link: Storno, backLink: ReversedBy, negate: [debit, credit] }
```

## Expected behaviour

- A created document is **retired** when its status seed is classified `cancelled` or `void`, when it
  is the twin of a `reversals` action on the target (its `link` is set), or when its own twin (its
  `backLink`) stands in the reversal's `when` status.
- On every occurrence of the moment - a re-issue, or the update of an `onCreate` posting's source -
  retired documents are not "the" post: a live draft is rewritten in place, a live posted entry that
  nobody reversed is reported and left alone, and when every document found is retired a new draft is
  created from the current figures.
- Retired documents accumulate over repeated rejections; the live one is the one compared.
- A `reverses` sibling reverses the live original, and counts as its own only the reversals of that
  original, so the twin of an earlier round is not read as its storno.

## Edge rules

- An `onCreate` posting's update still never writes a first post: a source with no created document
  at all belongs to the create. Only a source whose every document is retired gets a new draft from
  an update.
- A target whose status seeds declare no `stage` cannot tell a rejected entry from a posted one; only
  the reversal half of the rule applies to it.
- A target without `reversals` and without a `stage` classification behaves as in 1.11.

## Prior art / workarounds

None within the file. The edit was reported with "correct it with a reversing entry", although the
entry it named was already rejected or reversed, and the corrected source stayed off the ledger.
Reference implementation: https://github.com/eclipse-dirigible/dirigible/pull/8438.

## Specification text

**Anchor:** Declarative glue > postings - source document to ledger > An amended source rewrites its
posting, appended after the **Normative** block for `onCreate` (the one 0075 added).

The post the comparison reads is the source's **live** one. A back-reference found on a rejected
entry, or on an entry reversed in the ledger, or on the reversing twin itself, does not represent the
source any more, and a source whose every entry is dead is simply not on the ledger.

> **Normative.** A created document whose status seed is classified `stage: cancelled` or `void`, or
> that is the twin of a [`reversals`](#reversals--a-posted-documents-negated-twin) action on the
> target, or whose own twin stands in the reversal's `when` status, is **retired**. On every
> occurrence of a posting's moment, an `onCreate` posting's source update included, a conforming
> generator MUST ignore retired documents when it looks for the post the `backReference` finds: a
> live document still in its create status MUST be rewritten in place, a live document that has
> left it MUST be left alone and the divergence reported, and when every document found is retired a
> new draft MUST be created from the current figures. A `reverses` sibling MUST reverse the live
> original and MUST count as its own only the reversals of that original. A target whose status
> seeds declare no `stage` cannot be told rejected from posted, and only the reversal half applies.

This amends the "Once the created document has left that status" sentence above and 0075's rule
that an update "MUST NOT create a post from an update where none exists": a source whose every
created document is retired has none that represents it, and its update creates one; a source with no
created document at all still belongs to the create.
