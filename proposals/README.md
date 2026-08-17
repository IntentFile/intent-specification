# Proposals

Every semantic change to the format lives here first, and lives here alone until a release
folds it into a version document. A version document is written once, at a release, and never
again — so this directory is where the format's pending meaning is kept, in the open.

Copy [`0000-template.md`](0000-template.md) to `NNNN-short-name.md` (next free number), fill it
in, and open a pull request. **A proposal PR never touches `versions/`.**

## Status lifecycle

| Status | Means |
| --- | --- |
| `draft` | open for discussion; the PR is not merged yet |
| `accepted` | the direction is agreed and the proposal PR is merged — **not published**; it waits for a release |
| `released in <version>` | folded into that version document; the proposal stays as the record of why |
| `rejected` | decided against, with the reasoning kept in the file |
| `withdrawn` | the author took it back, or the gap was closed another way |

A proposal is never deleted. A rejected one is the most useful kind to find later — it says why
the obvious idea was not taken.

## What a proposal must carry

Beyond the problem, the shape and the expected behaviour, an accepted proposal has to be
**foldable by someone who was not in the discussion**:

- **`Specification text`** — the exact prose that will go into the version document, written as
  specification, not as argument. This is what the release copies in.
- **`Anchor`** — where it goes: the chapter or section it follows, so two proposals folded on
  the same day cannot land in the wrong order.
- **`DSL index`** — the *Appendix A* row the construct gets, if it adds one.

A proposal without those is a discussion, not a proposal, and the release that has to invent
the wording is the release that gets it wrong.

## Smaller changes

An [issue](https://github.com/IntentFile/intent-specification/issues/new/choose) is the right
place to report a gap — it commits the project to nothing. Editorial fixes need neither an issue
nor a proposal: open a PR against the current version document. See
[CONTRIBUTING.md](../CONTRIBUTING.md).
