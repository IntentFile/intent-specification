# A self-reference that stays a tree - `tree: true`

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7988
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8823

## The problem

A `manyToOne` onto its own entity is how a model says "tree": a task's parent task, a cost centre's
parent cost centre, an organisational unit's parent unit. Nothing keeps it one. A row can be saved as
its own parent, or under one of its own descendants, and every walk up the edge afterwards - a tree
view, a roll-up over the levels, a breadcrumb, a "everything under this node" filter - loops.

[`hierarchy:`](#hierarchy--leafonly--tree-entities) closes the gap, but it brings the tree-table
list and the leaf-only machinery with it, which a task's parent task does not want. No `checks:` kind
can say "not my own ancestor": the rule reads other rows, transitively.

Two concurrent moves make it worse: A moved under B and B moved under A, each checked against the
data as it stood before the other, both pass, and the cycle is closed by two writes that were each
valid on their own.

## The proposed shape

```yaml
- name: ProjectTask
  relations:
    - { name: Parent, kind: manyToOne, to: ProjectTask, tree: true,
        treeMessage: "A task cannot sit under itself" }
```

## Expected behaviour

- A create or update whose `tree: true` relation names the row itself, or one of the row's
  descendants, is refused as invalid input naming the relation. `treeMessage:` replaces the
  implementation's refusal text; it is seeded in the translation catalog like a check message.
- The check holds under concurrency: two writes that would close a cycle together never both
  commit.
- A cycle already in the data (written before the rule existed) ends the walk; it does not loop
  it.
- The [`hierarchy:`](#hierarchy--leafonly--tree-entities) edge is a tree edge without declaring
  `tree: true`, and is held to the same rule, on every writer of the entity rather than only on the
  generated user interface.

## Edge rules

- `tree: true` is valid only on a plain, optional, non-composition `manyToOne` whose target is the
  declaring entity itself, in the same model. Any other relation is an authoring error: another target
  has no ancestors of this row, a composition parent is preset rather than picked, and a required edge
  leaves no way to create a root.
- `treeMessage:` without `tree: true`, and a blank `treeMessage:`, are authoring errors.
- An update is asked only when it changes the relation.
- A model declaring no `tree:` generates as before.

## Prior art / workarounds

`hierarchy:` on the entity, accepting the tree-table list it brings; otherwise hand-written
validation in the escape-hatch folder.

## Specification text

**Anchor:** Entities & fields > `hierarchy / leafOnly — tree entities`, after the paragraph that
follows the example.

A self-reference that is a tree but wants none of the tree list - a task's parent task - declares the
edge alone:

```yaml
- name: ProjectTask
  relations:
    - { name: Parent, kind: manyToOne, to: ProjectTask, tree: true,
        treeMessage: "A task cannot sit under itself" }
```

The edge never closes a cycle: a row cannot be saved under itself or under one of its descendants.
The `hierarchy:` edge is such an edge already, without the key.

> **Normative.**
> For a relation declaring `tree: true`, and for the relation an entity names as its `hierarchy:`, a
> conforming generator MUST refuse a create or an update whose value for the relation is the row
> itself or one of its descendants, as invalid input naming the relation and carrying `treeMessage:`
> where one is declared, or the implementation's message otherwise; the message is seeded in the
> translation catalog. The refusal MUST hold for every writer of the entity, not only for a generated
> user interface, and MUST hold under concurrency: two writes that would together close a cycle MUST
> NOT both commit. A cycle already present in the stored data MUST end the walk rather than loop it.
> An update MUST be asked only when it changes the relation. `tree: true` on anything but an optional,
> non-composition `manyToOne` whose target is the declaring entity in the same model, `treeMessage:`
> without `tree: true`, and a blank `treeMessage:` MUST be reported as authoring errors. A model
> declaring no `tree:` MUST generate byte-identical output.

The paragraph after the `hierarchy` example ("the server rejects cycles ...") keeps its meaning; the
cycle half of it is now stated by the block above.

## DSL index

| Construct | What it does |
| --- | --- |
| [`tree`](#hierarchy--leafonly--tree-entities) / `treeMessage` | a same-entity `manyToOne` that never closes a cycle, without the tree list of `hierarchy` |
