# An unrecognised key is an authoring error, never ignored


- **Status:** released in [1.5](../versions/1.5.md)
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/6541
- **Discussion:** https://github.com/IntentFile/intent-specification/pull/24

A typed mapping normally drops a key it does not know. That silence is the worst failure this format can have: the file is accepted, generation succeeds, the application deploys, and the only symptom is that the promise the author wrote is absent at runtime — with every step of the pipeline reporting success. A case slip (`Required:` for `required:`) is the same failure and the hardest to see by eye.

The format already demands this honesty of generators elsewhere ("report what you cannot resolve rather than ignore it"); this states it for the vocabulary itself.

**Authoring rules** gains a bullet and an *Unrecognised keys* rule:

> A conforming generator MUST report a key it does not recognise as an authoring error rather than ignoring it, and the report MUST name the key, where it appears, and — where one exists — the nearest declared name. Key names are case-sensitive […] This applies equally to a seed row, whose keys are the target entity's own names rather than this specification's. A map whose keys are drawn from the model being described (a `map:` projection, a relation's `where:`, a widget's `at:`) is validated against that model, not against this vocabulary.

**Seeds** — the existing normative rule is sharpened rather than replaced: a row key is a field name, a **to-one** relation name (a collection has no column to set) or the `stage` marker, and what accepting anything else silently costs is now spelled out — a dropped NOT NULL foreign key makes the import skip *every* row, so a nomenclature lands as zero rows behind a fully green pipeline.

No new construct, so Appendix A is unchanged.

Proven out in the reference implementation: eclipse-dirigible/dirigible#6748 (raw-tree key validation against the model classes + seed-row keys), verified against 63 real production intent files with no false positives. Motivating report: eclipse-dirigible/dirigible#6541.

## Specification text

The prose below is what a release folds into the next version document, at the anchors
given. It was written against `versions/1.2.md` and is carried here unchanged.

**Anchor:** Overview > Authoring rules

- **Only the keys this specification declares exist, and they are case-sensitive.** An invented key, or a case slip (`Required:` for `required:`), is an authoring error - never a key that is accepted and ignored.

#### Unrecognised keys

A typed mapping normally drops a key it does not know. That silence is the worst failure this format can have: the file is accepted, generation succeeds, the application deploys, and the only symptom is that the promise the author wrote is absent at runtime - with every step of the pipeline reporting success. The rule is therefore the same one the format applies to a reference it cannot resolve.

> **Normative.**
> A conforming generator MUST report a key it does not recognise as an authoring error rather than
> ignoring it, and the report MUST name the key, where it appears, and - where one exists - the
> nearest declared name. Key names are **case-sensitive**: a key differing from a declared one only
> in case is unrecognised, and the report SHOULD say so, since it is the slip hardest to see by eye.
> This applies equally to a [seed row](#seeds), whose keys are the target entity's own names rather
> than this specification's. A map whose keys are drawn from the model being described (a `map:`
> projection, a relation's `where:`, a widget's `at:`, a delegate's injected `fields:`) is validated
> against that model, not against this vocabulary.

Being written as a mapping does not make a block free-form. A process [`trigger:`](#processes), an
[`abortOn:`](#aborton--cancel-the-instance-on-a-terminal-status), a glue [`event:`](#notifications)
binding, a step's [`args:`](#processes) and the blocks nested inside them are each a fixed
vocabulary, and a key outside it is unrecognised like any other.

> **Normative.**
> A step's `args:` are recognised **per step kind**: an argument declared on a kind that does not
> read it (a decision's `if` on a user task, a boundary `timeout` on a service task) MUST be reported
> like an unrecognised one, and the report SHOULD name the kind that does read it. This is the same
> failure and not a lesser one - the step reads nothing, so the argument does nothing.

**Anchor:** Data, seeds & naming > seeds

> Row keys must match a field name, a **to-one** relation name (a collection has no column to set), or the `stage` marker below, **exactly** (case-sensitive). A key matching none of those is an authoring error, reported with the nearest declared name — see [unrecognised keys](#unrecognised-keys). Accepting it would drop the column, and a dropped NOT NULL foreign key makes the import skip **every** row: a nomenclature that imports as zero rows, behind a fully green pipeline.
