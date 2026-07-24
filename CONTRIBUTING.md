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
| Change proposals under discussion | `proposals/` |
| Complete example `.intent` files | `examples/` |
| The rendered website | the separate [intentfile.github.io](https://github.com/IntentFile/intentfile.github.io) repository |

## Two kinds of change

### 1. Editorial changes — just open a PR

Typos, wording, broken links, clearer examples, better cross-references — anything that
does **not** change what a conforming file or generator must do. Open a pull request
directly against the current version document; no prior issue needed.

### 2. Specification changes — propose first

Anything that changes the meaning of the format: a new construct or attribute, new allowed
values, changed semantics, deprecations. Start with an
[issue](https://github.com/IntentFile/intent-specification/issues/new/choose) — or, for a
larger design, a document under `proposals/` (copy `proposals/0000-template.md`) —
describing:

- **The problem** — what cannot be expressed today, with a concrete scenario.
- **The proposed shape** — the YAML an author would write.
- **The expected behaviour** — what a conforming generator must produce, stated
  platform-neutrally.
- **Prior art** — how the scenario is handled today (hand-written code, a workaround,
  another format).

Once the direction is agreed, open a PR that updates the current version document —
including the *Appendix A: DSL index* row — and adds or updates an example when the
construct benefits from one. A construct without at least one implementation proving it
out is normally listed as *planned* rather than specified as required.

## Ground rules for spec text

- **Vendor-neutral, always.** The specification describes the format and the observable
  behaviour of a conforming generator — never a particular product, runtime, or package
  layout. If a sentence only makes sense for one implementation, it does not belong here.
- **Normative vs. informative.** Rules a conforming file or generator MUST follow are
  marked **Normative**; everything else is guidance.
- **Every construct earns its place** with a realistic YAML snippet, its attributes, and
  its edge rules — mirror the structure of the existing chapters.
- **Keep the index in sync.** A new construct gets a row in *Appendix A: DSL index*;
  changed semantics update the chapter and the row together.

## Versioning

Editorial fixes and clarifications land on the current version document. Semantic changes
are batched: they accumulate through merged proposals and are released as a new
`versions/<next>.md`, leaving previous versions immutable. The website renders the current
version.

## Review and merging

Maintainers review every PR. Editorial changes merge on one approval; specification
changes merge when the discussion in the linked issue or proposal has converged and a
maintainer approves.
