# The scope boundary, and the authoring assistant's honesty

- **Status:** draft
- **Issue:** [IntentFile/intent-specification#11](https://github.com/IntentFile/intent-specification/issues/11)
- **Proposed for:** version 1.1 (see `versions/1.1.md` in this pull request)

## The problem

The specification states the altitude contract ("the intent never emits application code; it stops
at the model layer — that boundary is non-negotiable"), and the Guardrails say the escape hatch is
non-negotiable ("curated vocabulary, not a general DSL; real logic is a `script` step or a
hand-written hook"). What it never says is **which kinds of requirement live beyond the boundary
and why** — so every author, and every AI assistant proposing patches, rediscovers the line ad hoc.

That gap has a concrete failure mode, observed in practice. An assistant asked for a requirement
the format cannot express tends to **silently substitute the nearest expressible thing**: asked for
"automatically identify X from a register and automatically create document Y", it proposed a
manual user task and a manual button — a proposal that parses and validates completely clean while
quietly changing the application's contract from "the system does it" to "a person does it". A
stated requirement (a history log) was dropped without a word. The reviewer has no signal that
anything is missing, and the format's maintainers never learn which constructs real projects
actually lacked.

## The proposed shape

Two additions to the Overview chapter, one informative and one normative.

**1. An informative section — "The scope boundary".** Three recurring categories of requirement
are *how*, not *what*, and are deliberately outside the format — each with its designated hand-off:

| Beyond the boundary | Why it is not intent | The hand-off |
| --- | --- | --- |
| **Protocol adaptation** — certificates, acknowledgments, retries, batch/file transports | `integrations` / `inbound` are one-line call-outs by design | an integration route in the platform's integration technology |
| **Algorithms** — checksums, fuzzy matching, scoring, policy tie-breaks | the format draws this line for `pattern` already: a format check, not a semantic one | a calculated-field call-out or a service-task `delegate`, hand-written in the custom folder |
| **Statutory / designed form** — the exact mandated print layout | the print template is create-if-absent *by design* | the authored template itself |

The section also states the framing: the boundary is a feature. Everything inside it is
deterministic, regenerable and reviewable; everything outside it enters through a first-class,
documented hand-off instead of a workaround. Partial expressibility honestly stated is more
trustworthy than pretended total coverage.

**2. A normative conformance rule for authoring assistants.** The overview names "an AI assistant
proposing patches" as a first-class author at altitude 1, yet the specification places no
obligation on it — while generators are already bound by rules of the same class ("MUST be
reported as an authoring error rather than ignored"). Proposed:

> **Normative.**
> An authoring assistant that cannot express a requirement in this format MUST say so rather than
> silently substituting weaker semantics — a manual step proposed where automation was requested is
> a changed contract, not a smaller change. It MUST NOT drop a stated requirement from a proposal
> without reporting it. It SHOULD name the category of the gap and the designated hand-off point,
> and it MUST NOT imply that hand-off code will be generated when it is the developer's to write.

## Expected behaviour

An assistant facing an out-of-scope requirement produces a proposal that is **maximally
intent-driven with a clearly marked hand-written remainder**: the intent-side wiring (the step, the
field, the button) is proposed, the hand-off point is named, and the reply states plainly which
part is the developer's to write and why the format does not carry it. A requirement that is plain
modeling the format *does* support (a log entity is just an entity) is never dropped — omitting it
is a proposal-quality defect, not a scope question.

## Edge rules

- The rule binds the **assistant's report**, not its creativity: proposing a *better* modeling of
  the same contract is welcome; proposing a *weaker contract* without saying so is what is
  forbidden. The test is whether the application still does what was asked without a human doing
  part of it by hand.
- "Say so" is part of the proposal's deliverable, not a log line: the statement must reach the
  reviewer in the same reply that carries the proposal.
- The rule is deliberately implementation-neutral: it does not prescribe a reply format, a UI
  surface, or a feedback channel — only that the gap is reported, categorised where possible, and
  never papered over.
- The section must not grow into a capability matrix of any one platform. It names *categories*
  and *hand-off kinds*; a platform documents its own concrete technologies (which routes, which
  folder, which override switch) in its own guides.

## Prior art / workarounds

The Guardrails bullet ("the escape hatch is non-negotiable") has carried the whole weight of this
topic since 1.0 — one line, addressed to generator authors, with nothing for the authoring layer.
Conventional code assistants ship "capability cards" / system-prompt scope statements for the same
reason: an assistant that knows its envelope degrades gracefully at the edge, one that does not
substitutes silently. The observed manual-task substitution above is the workaround in the wild
today: it costs nothing at authoring time and is paid for at acceptance, when the "automated"
process turns out to have a person inside it.
