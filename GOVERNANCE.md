# Governance

The Intent File Specification is developed in the open under a lightweight,
maintainer-led model.

## Roles

- **Contributors** — anyone who opens an issue, a proposal, or a pull request.
- **Maintainers** — the people with merge rights on this repository. Maintainers steward
  the specification's coherence and its vendor-neutrality, review proposals and PRs, and
  cut versions.

New maintainers are added by consensus of the existing maintainers, based on a sustained
record of quality contributions.

## Decision making

- Day-to-day decisions (editorial merges, issue triage) are made by any maintainer.
- Specification changes are decided as **proposals**: a converged discussion on the proposal
  pull request and a maintainer's approval; when maintainers disagree, they resolve it by
  simple majority. Merging a proposal records acceptance — it publishes nothing.
- A new specification **version** is cut by maintainer consensus once the accepted proposals
  warrant it. The release folds in every accepted proposal and nothing else. Version documents
  are immutable: a release adds a file and never edits an earlier one.

## Guiding constraints

Two constraints outrank any feature request:

1. **Vendor-neutrality.** The specification must never require, name, or structurally
   favour a particular product, runtime, or implementation.
2. **The altitude boundary.** The Intent File describes intent and stops at the model
   layer; a change that would have it emit or prescribe application code is out of scope
   by definition.
