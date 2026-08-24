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
| [1.4](versions/1.4.md) | superseded - added the glue event axis: process-step events and queue/topic/folder arrivals ([proposal](proposals/0012-glue-event-axis.md)), `attach: recordPrint` fan-out ([proposal](proposals/0011-record-print-fan-out.md)), notify link placeholders `{recordUrl}` / `{inboxUrl}` / `{appUrl}` ([proposal](proposals/0013-notify-link-placeholders.md)), and role-scoped field visibility `visibleTo` ([proposal](proposals/0004-role-scoped-field-visibility.md)) |
| [1.5](versions/1.5.md) | superseded - adds unrecognised-key errors ([proposal](proposals/0014-unknown-keys-are-errors.md)), multilingual report columns ([proposal](proposals/0015-multilingual-report-columns.md)), the `related` register ([proposal](proposals/0016-related-registers.md)), the declared `payload` envelope ([proposal](proposals/0017-declared-payload.md)), `outbound` departures ([proposal](proposals/0018-outbound-departures.md)), relative moments in a schedule's `where` ([proposal](proposals/0019-schedule-relative-moments.md)), and entity-level `unique` ([proposal](proposals/0020-entity-level-unique.md)) |

| [1.6](versions/1.6.md) | current - adds report `parameters` ([proposal](proposals/0021-report-parameters.md)), `kind: statement` financial statements ([proposal](proposals/0021-financial-statement-definitions.md)), the `subset` relation ([proposal](proposals/0026-subset-relation.md)), print placeholder fallbacks `{{a|b}}` ([proposal](proposals/0027-print-placeholder-fallback.md)), `fileName` patterns ([proposal](proposals/0026-naming-a-rendered-document.md)), step-axis + `mode: append` create-froms ([proposal](proposals/0021-generates-step-axis-and-append.md)), the retired-target guard ([proposal](proposals/0024-supersede-a-retired-target.md)), `sourceStatusOnRetire` ([proposal](proposals/0025-declared-reopen-on-retire.md)), and `inbound` `accept`/`map` ([proposal](proposals/0021-arrival-mapping.md)); states precisely: roll-up update/re-parent recompute ([proposal](proposals/0021-rollups-recompute-on-child-update.md), [proposal](proposals/0022-rekey-repairs-both-groups.md)), settlement recompute on correction ([proposal](proposals/0021-settlement-recompute-on-correction.md)), and expansion reconcile + master delete ([proposal](proposals/0023-expansion-reconciles-its-child-set.md), [proposal](proposals/0021-expansion-master-delete.md)) |

## Structure

```
versions/     the specification, one file per version - the newest version file is normative (currently versions/1.6.md)
              written once, at a release, and never edited again
proposals/    every pending semantic change, in the open, until a release folds it into a version
examples/     complete example .intent files
```

A version document does not change between releases: cite `versions/1.2.md` today and it reads
the same next month. What the format is *about to* become lives in `proposals/`.

## Participate

The specification evolves in the open — every change is a pull request against this
repository:

- **Editorial fixes** (typos, wording, clearer examples): open a PR directly against the
  current version document.
- **Specification changes** (new constructs, changed semantics): report the gap as an
  [issue](https://github.com/IntentFile/intent-specification/issues/new/choose), then open a PR
  adding a document under `proposals/`. That PR never touches `versions/` — merging it records
  that the direction is **accepted**, and the change is published when a release folds every
  accepted proposal into a new version.

See [CONTRIBUTING.md](CONTRIBUTING.md) for the full flow and the ground rules
(vendor-neutrality above all), and [GOVERNANCE.md](GOVERNANCE.md) for how decisions are
made. Participation is covered by the [Code of Conduct](CODE_OF_CONDUCT.md).

## License

[Apache License 2.0](LICENSE)
