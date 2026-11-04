# A call: step naming a script handler is deprecated; a call: naming a class runs as delegate: does

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/162, https://github.com/eclipse-dirigible/dirigible/issues/8009
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8019

## The problem

1.11 names `call` beside `delegate` without saying what it binds. The Service tasks section lists
the shapes `setField` / `clearField` / `setRelationField`, `notify` and `delegate`; `call` appears
only in the rules that exclude it - "it MUST NOT carry another action (`setField`, `clearField`,
`setRelationField`, `call`, `delegate`)" - and in the `retry` / `onError` note, "on a `call:` or bare
service task, because neither is covered".

A reader cannot tell which of the two is the format's construct for a step's hand-written work. In
the reference implementation a `call:` could name either a class or a script handler; a script
handler needs a script runtime beside the application for a single step, and a new file written
against it is written against a form the implementation is moving away from.

## The proposed shape

```yaml
steps:
  # the construct: hand-written work is a delegate
  - { name: archive,  kind: serviceTask, args: { delegate: ArchiveInvoice, next: done } }
  # read as `delegate: ArchiveInvoice`
  - { name: archive2, kind: serviceTask, args: { call: ArchiveInvoice, next: done } }
  # names a script handler: deprecated, reported on every generation, to be rewritten as a delegate
  - { name: legacy,   kind: serviceTask, args: { call: scripts/archiveInvoice, next: done } }
```

## Expected behaviour

A `call:` that names a class runs exactly as a `delegate:` naming that class does. A `call:` that
names a script handler keeps working, and is reported as deprecated on every generation, the report
naming the process, the step and the handler. The report is advisory, not an authoring error: the
rewrite needs the class written first, which no change to the file alone can provide.

## Edge rules

- The `call:` exclusions stay: a `notify` step and a `delegate:` step MUST NOT also carry a
  `call:`.
- `retry` / `onError` stay refused on a `call:` step, whatever it names; a step that needs them is
  written as a `delegate:`.
- A file that uses only `delegate:` is unaffected.

## Prior art / workarounds

None needed: this states what the format already intends. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8019.

## Specification text

**Anchor:** Processes & forms > processes > Service tasks, appended to the paragraph
"Service-task shapes: ..." (before the `setField` paragraph).

A service task's hand-written work is a **`delegate:`**. A `call:` names the same work in an older
spelling: a `call:` that names a class is read as a `delegate:` of that class, and a `call:` that
names a script handler is **deprecated**.

> **Normative.** A conforming generator MUST treat a `call:` naming a class exactly as a `delegate:`
> naming that class. A `call:` naming a script handler is deprecated: a generator that still accepts
> it MUST report it on every generation, naming the process, the step and the handler, as an
> advisory rather than an authoring error, and an author SHOULD rewrite it as a `delegate:`.

## DSL index

No new row.
