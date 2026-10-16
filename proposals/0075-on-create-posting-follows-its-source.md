# An `onCreate` posting follows its source's edit and delete

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/111, https://github.com/eclipse-dirigible/dirigible/issues/7634
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7673
- **Amends:** [0032 - an amended source rewrites its posting](0032-amended-source-rewrites-its-posting.md)

## The problem

0032 describes the rewrite for a posting whose moment fires again - a document issued, rejected,
amended, issued again. An `onCreate` posting, the trigger for a source with no status lifecycle - a
booked payment - fires once, at the insert, and never again. An edit of the payment leaves the DRAFT
journal entry with the insert's figures, and `reverses:` is bound to transitions only, so a deleted
payment's posted entry goes on booking money with no storno. The specification leaves both open, and
"nothing happens" is what a reader infers.

## The proposed shape

No new key. The behaviour applies to every posting declared with `event: { onCreate: <Entity> }`.

## Expected behaviour

An `onCreate` posting follows its source:

- **Edit.** An update of the source re-derives the content and applies 0032's comparison: while the
  created document still holds its `init:` status it is rewritten; once it has left it, the
  divergence is reported and the correction is left to `reverses`. A create and an edit committed as
  one unit post once.
- **Delete.** A delete of the source reverses the created document through `reverses:` when the
  posting declares one, as a transition into a reversing status would; without `reverses:` the
  divergence is reported.

## Edge rules

- The twin reaction never writes a first post: an update that finds no post does nothing, since the
  create's own reaction is what posts.
- A source that is deleted while its post is still rewritable is reversed, not deleted: a posted
  figure, even a draft one, leaves a trail.

## Prior art / workarounds

A hand-written listener on the source's update and delete. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/7673.

## Specification text

**Anchor:** Declarative glue > postings > An amended source rewrites its posting, appended.

A posting bound to `onCreate` has no second occurrence of its moment, so its source's **update** is
that occurrence: the content is re-derived and compared exactly as above. Its source's **delete** is
a reversal: where the posting declares `reverses`, the created document is reversed as it would be on
a reversing transition.

> **Normative.** For a posting whose event is `onCreate`, a conforming generator MUST apply the
> derive-and-compare rule above on every update of the source, MUST NOT create a post from an update
> where none exists, and MUST treat a create and an update committed in one unit of work as one
> occurrence. On the delete of the source it MUST reverse the created document through `reverses`
> where one is declared and MUST report the divergence where none is.

## DSL index

No new row; the `postings` row is unchanged.
