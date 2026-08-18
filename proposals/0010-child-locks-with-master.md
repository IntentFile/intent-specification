# An entity's immutability covers the collections composed into it


- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/6695
- **Discussion:** https://github.com/IntentFile/intent-specification/pull/19

1.2 already made the default normative — "`locksWithMaster` defaults to **true**, so a child that says nothing keeps freezing with its master" — but it only ever spelled out what *freezing* means for the child declared `locksWithMaster: false`, where it is normative that the affordances must stay alive and not merely the writes. The default was left to be read as the affordances alone.

One implementation read it exactly that way: the child's own endpoint went on accepting creates, edits and deletes against a locked master, while its UI withheld them.

That is not a cosmetic gap. A child write maintains the master's derived values — a line resums the document's totals — so it reaches precisely what the lock protects, after the number was stamped, the frozen copy taken and the ledger entry posted from those totals. The lock is undone through a different door, and the document's own page then shows a total that disagrees with the copy it printed.

This says it in both directions instead: the default freezes the child's writes AND its affordances, the opt-out reopens both, and system / workflow writes stay possible throughout — they are what corrects an immutable record. No new keyword; the `immutableWhen` section gains the normative paragraph and a cross-reference, and the `locksWithMaster` section's opening no longer claims a locking master "says nothing" about its children.

Implementation that motivated it: [eclipse-dirigible/dirigible#6739](https://github.com/eclipse-dirigible/dirigible/pull/6739).

🤖 Generated with [Claude Code](https://claude.com/claude-code)

## Specification text

The prose below is what a release folds into the next version document, at the anchors
given. It was written against `versions/1.2.md` and is carried here unchanged.

**Anchor:** stamped at a modeled issue step (a placeholder holds the field until then): > immutableWhen / immutable — user-write immutability

An entity's immutability also covers its **composition children**, which declare none of their own and whose writes maintain the master's derived values — see [`locksWithMaster`](#lockswithmaster--a-child-collection-that-outlives-its-masters-lock) for the collection that must outlive the lock.

> **Normative.**
> A generator MUST refuse a user create, update or delete of a composition child whose master is
> currently immutable, unless that child declares `locksWithMaster: false`. The refusal MUST cover
> every user surface it generates, not only the affordances it renders: a child write that
> maintains the master's derived values — a line that resums the document's totals — reaches
> exactly what the lock protects, and permitting it undoes the lock through a different door. It
> MUST NOT extend to system / workflow writes, which are what corrects an immutable record.

**Anchor:** stamped at a modeled issue step (a placeholder holds the field until then): > locksWithMaster — a child collection that outlives its master's lock

An entity's immutability covers that entity **and the collections composed into it**. For some
children that default is wrong — a master that freezes its own content says nothing about a
collection recording what happens to the document afterwards:

**Anchor:** stamped at a modeled issue step (a placeholder holds the field until then): > locksWithMaster — a child collection that outlives its master's lock

> master — in the affordances a generator renders for that collection AND in the writes it accepts
> for it (see [immutability](#immutablewhen--immutable--user-write-immutability)).
