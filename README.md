# The Intent File Specification

The **Intent File Specification** defines a vendor-neutral format — a single declarative
YAML file (`*.intent`) per project — that describes a whole application one altitude above
the models a conforming generator produces from it: its entities, relations, workflows,
forms, reports, permissions, integrations and seed data.

The Intent File never emits application code. It stops at the model layer; a conforming
generator deterministically turns it into the platform's model artefacts, and those into a
complete running application.

This repository is the **normative home of the specification**. A rendered, navigable
version — with the DSL reference, worked examples and the Intent-Driven Development
manifesto — is published at **[intentfile.org](https://intentfile.org)** (source:
[intentfile.github.io](https://github.com/IntentFile/intentfile.github.io)).

## Current version

| Version | Status |
| --- | --- |
| [1.0](versions/1.0.md) | current |
| [1.1](versions/1.1.md) | proposed - adds `aggregates`, `checks: kind: guard` and `posts` ([proposal](proposals/0001-keyed-aggregates-guard-checks-and-row-posting.md)) |

## Structure

```
versions/     the specification, one file per version - versions/1.0.md is normative
proposals/    proposals for specification changes, discussed before they are specified
examples/     complete example .intent files
```

## Participate

The specification evolves in the open — every change is a pull request against this
repository:

- **Editorial fixes** (typos, wording, clearer examples): open a PR directly.
- **Specification changes** (new constructs, changed semantics): start with an
  [issue](https://github.com/IntentFile/intent-specification/issues/new/choose) or a
  document under `proposals/`, then a PR once the direction is agreed.

See [CONTRIBUTING.md](CONTRIBUTING.md) for the full flow and the ground rules
(vendor-neutrality above all), and [GOVERNANCE.md](GOVERNANCE.md) for how decisions are
made. Participation is covered by the [Code of Conduct](CODE_OF_CONDUCT.md).

## License

[Apache License 2.0](LICENSE)
