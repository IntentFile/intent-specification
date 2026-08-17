# Contributing to the Intent File Specification

Thank you for your interest! The specification evolves in the open: every change — from a
typo fix to a new DSL construct — is proposed, discussed and merged as a **pull request**
against this repository.

By participating you agree to our [Code of Conduct](CODE_OF_CONDUCT.md). Contributions are
accepted under the repository's [Apache 2.0 license](LICENSE).

## What lives where

| Content | Where |
| --- | --- |
| The normative specification | `versions/<version>.md` (the current version is listed in the README) |
| Change proposals — every semantic change, before it is released | `proposals/NNNN-*.md` |
| Complete example `.intent` files | `examples/` |
| The rendered website | the separate [intentfile.github.io](https://github.com/IntentFile/intentfile.github.io) repository |

## The one rule that shapes everything else

**A version document is written once, at a release, and never again.** Between releases the
specification does not change: a reader who cites `versions/1.4.md` today reads the same
sentences next month. Everything that will change the meaning of the format waits in
`proposals/`, in the open, where it can be read, discussed and counted — and lands in one
deliberate act when a new version is cut.

That gives the format two clocks, deliberately: proposals move continuously, the
specification moves in releases.

## Two kinds of change

### 1. Editorial changes — a direct PR against the current version

Typos, wording, broken links, clearer examples, better cross-references — anything that
does **not** change what a conforming file or generator must do. Open a pull request
directly against the current version document; no proposal, no prior issue needed.

The rule above is about *meaning*, not about characters: correcting a sentence that says
the wrong thing is fixing the published version, not changing it. If you find yourself
arguing that a change is editorial, it is not — write a proposal.

### 2. Semantic changes — a proposal, and only a proposal

Anything that changes the meaning of the format: a new construct or attribute, new allowed
values, changed semantics, deprecations, a normative rule.

1. **Start with an [issue](https://github.com/IntentFile/intent-specification/issues/new/choose)**
   describing the gap — the problem, a concrete scenario, and how it is handled today. An
   issue is a report; it commits the project to nothing.
2. **Open a PR that adds `proposals/NNNN-short-name.md`** (copy
   [`proposals/0000-template.md`](proposals/0000-template.md), take the next free number).
   Discussion happens on that PR.
3. **That PR MUST NOT touch `versions/`.** A pull request that changes both a proposal and a
   version document is sent back — those are two different acts, separated on purpose. The
   proposal carries the exact specification text it will contribute, and the anchor it belongs
   under, so the release that folds it in is mechanical rather than a rewrite.
4. Merging the proposal PR means the direction is **accepted** — it does not publish anything.
   The proposal's `Status` becomes `accepted`, and it waits for a release.

A construct is normally proven by at least one implementation before a release specifies it as
required; until then it may be released as *planned*, or wait. An implementation MAY ship a
construct before the specification releases it — in that window the accepted proposal is the
citable reference, and the implementation's own documentation says so.

## Releasing a version

A release is one pull request, opened when the maintainers decide there is enough to release:

1. Copy the current `versions/<current>.md` to `versions/<next>.md`.
2. Fold in **every proposal whose status is `accepted`**, each at the anchor its proposal names.
   Nothing else goes in — an unaccepted proposal is not "nearly there", it is not in.
3. Update *Appendix A: DSL index* for every construct folded in, and add or update the examples
   the new constructs benefit from.
4. Flip each folded proposal's status to `released in <next>` and link the version.
5. Update the README's version table: the new version becomes current, the previous one
   `superseded`.
6. Leave every earlier `versions/*.md` **byte-identical**. A release adds a file; it never
   edits one.

After the merge, the website repository renders the new current version. Between releases the
site is stable, because the specification is.

## Ground rules for spec text

- **Vendor-neutral, always.** The specification describes the format and the observable
  behaviour of a conforming generator — never a particular product, runtime, or package
  layout. If a sentence only makes sense for one implementation, it does not belong here.
- **Normative vs. informative.** Rules a conforming file or generator MUST follow are
  marked **Normative**; everything else is guidance.
- **Every construct earns its place** with a realistic YAML snippet, its attributes, and
  its edge rules — mirror the structure of the existing chapters.
- **Keep the index in sync.** A new construct gets a row in *Appendix A: DSL index*;
  changed semantics update the chapter and the row together. Both happen at the release,
  from what the proposals carry.

## Review and merging

Maintainers review every PR. Editorial changes merge on one approval. A proposal merges when
the discussion has converged and a maintainer approves — which records acceptance, not
publication. A release PR merges when its fold has been read against each proposal it claims
to carry.
