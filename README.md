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
| [1.0](versions/1.0.md) | superseded |
| [1.1](versions/1.1.md) | superseded - added `aggregates`, `checks: kind: guard`, `posts`, the notify block (`attach: print`, `forEach`), `pattern` and header-mediated `dependsOn` ([proposal](proposals/0001-keyed-aggregates-guard-checks-and-row-posting.md)) |
| [1.2](versions/1.2.md) | superseded - added lifecycle stages + report `scope` + status names, `locksWithMaster`, `generates` prompted input and computed lines, `defaultValue` row seeding, print-template row filtering ([proposal](proposals/0002-print-template-row-filtering.md)), the scope boundary and the authoring assistant's honesty rule ([proposal](proposals/0003-the-scope-boundary.md)), and aligns document numbering with deployed practice |
| [1.3](versions/1.3.md) | superseded - added event-driven creation (`generates.event`, [proposal](proposals/0005-event-driven-generates.md)), `resolves` register lookup ([proposal](proposals/0006-register-lookup-resolves.md)), the `lifecycle` status graph ([proposal](proposals/0007-lifecycle-state-machine.md)), the `history` change trail ([proposal](proposals/0008-history-change-trail.md)), resolver-path task assignment ([proposal](proposals/0009-resolver-path-task-assignment.md)), and immutability covering composed collections ([proposal](proposals/0010-child-locks-with-master.md)) |
| [1.4](versions/1.4.md) | current - adds the glue event axis: process-step events and queue/topic/folder arrivals ([proposal](proposals/0012-glue-event-axis.md)), `attach: recordPrint` fan-out ([proposal](proposals/0011-record-print-fan-out.md)), notify link placeholders `{recordUrl}` / `{inboxUrl}` / `{appUrl}` ([proposal](proposals/0013-notify-link-placeholders.md)), and role-scoped field visibility `visibleTo` ([proposal](proposals/0004-role-scoped-field-visibility.md)) |

## Structure

```
versions/     the specification, one file per version - the newest version file is normative (currently versions/1.4.md)
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
