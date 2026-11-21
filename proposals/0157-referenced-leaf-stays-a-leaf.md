# A referenced leaf stays a leaf - `leafOnly` holds after the reference lands

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7826
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8826

## The problem

`leafOnly: true` makes sure a reference lands on a leaf of a
[`hierarchy:`](#hierarchy--leafonly--tree-entities) tree: a journal line books on account 4011,
never on the group 401 above it. Nothing keeps the leaf a leaf afterwards. A child created under
4011 - or a node re-parented under it - turns it into a group while the posted lines still point at
it. A balance report then shows a balance booked on a group on top of its children's roll-up, and
the sum of the leaves no longer equals the group.

The accounting rule is plain: an account with postings gets sub-accounts only after its balance has
been transferred. No key is needed to say it; it is what `leafOnly` already means.

## The proposed shape

No new key.

```yaml
- name: Account
  hierarchy: Parent
  relations:
    - { name: Parent, kind: manyToOne, to: Account }
# elsewhere, possibly in another model:
- { name: Account, kind: manyToOne, to: Account, model: accounts, leafOnly: true }
```

## Expected behaviour

- A create, or an update that changes the `hierarchy:` edge, whose new parent is referenced by any
  `leafOnly` relation is refused as invalid input naming the referencing entities and their number
  of rows. The message is seeded in the translation catalog.
- The count covers every `leafOnly` relation onto the tree, same-model and cross-model alike.
- The rule holds under concurrency: a child written under a node and a `leafOnly` reference written
  to the same node at once never both commit.

## Edge rules

- An update that does not change the parent is not asked.
- A `leafOnly` relation whose reference outlives its target by declaration (`whenTargetDeleted:
  keep`, proposed in [0126](0126-when-target-deleted-keep-and-reversal-on-delete.md)) does not
  count.
- A referencing model that is not deployed contributes nothing.

## Prior art / workarounds

A procedure outside the application ("transfer the balance first"), enforced by nobody.

## Specification text

**Anchor:** Entities & fields > `hierarchy / leafOnly — tree entities`, after the paragraph that
follows the example. Folded after [0156](0156-tree-self-reference.md) where both are accepted.

A reference that landed on a leaf keeps it one: a node that leaf-only references point at cannot gain
a child.

> **Normative.**
> A conforming generator MUST refuse a create of a row of a `hierarchy:` entity, and an update that
> changes its `hierarchy:` edge, whose new parent is referenced through any relation declaring
> `leafOnly: true`, as invalid input naming each referencing entity and the number of its rows, with
> a message seeded in the translation catalog. The count MUST cover every such relation whatever model
> declares it, and the rule MUST hold for every writer of the tree entity. A child and a leaf-only
> reference written to the same node concurrently MUST NOT both commit. An update that leaves the
> parent unchanged MUST NOT be asked.

## DSL index

No new row; the `hierarchy` / `leafOnly` row is unchanged.
