# Notify link placeholders - `{recordUrl}`, `{inboxUrl}`, `{appUrl}`


- **Status:** released in [1.4](../versions/1.4.md)
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/6553
- **Discussion:** https://github.com/IntentFile/intent-specification/pull/22

## Problem

A notification that cannot be acted on is a notification that gets ignored. The notify block interpolates fields into `subject` / `body`, but nothing in the specification lets a message carry the way back into the application — so "you have an approval waiting" arrives with no way to approve.

Nothing stops an author from pasting a route into the body, and that is the trap: the routes belong to whatever renders the application, so an intent that names one encodes a layout it does not own — correct only until that layout changes, and silently wrong afterwards.

## Change

`versions/1.2.md`, a new subsection under *The notify block* — **Links back to the application: `{recordUrl}`, `{inboxUrl}`, `{appUrl}`**:

| Placeholder | Resolves to |
| --- | --- |
| `{recordUrl}` | the record the message is about, opened in the application |
| `{inboxUrl}` | the recipient's task inbox |
| `{appUrl}` | the application's external base URL — the origin only |

Normative rules:

- The three names are **reserved** at every notify call site; an entity field of the same name MUST NOT shadow them.
- `{recordUrl}` and `{inboxUrl}` MUST be resolved by the implementation to a complete address. An intent MUST NOT be required to spell a route, and a conforming implementation MUST NOT require one. `{appUrl}` yields the origin alone — the escape hatch, with everything appended to it authored text.
- Inside a `forEach` fan-out `{recordUrl}` links the **ROW**, like every other bare path in the block; the anchor record stays addressable for values only.

Appendix A gains the index row.

## Reference implementation

eclipse-dirigible/dirigible#6744 — the resolver emits a bare identifier and reports the use, the model layer contributes only the entity and its key, and the code generator composes the address. That split is what the normative "an intent MUST NOT be required to spell a route" describes in practice.

Note the base `{appUrl}` origin token was never written up here (it shipped in the reference implementation as eclipse-dirigible/dirigible#6642), so this PR introduces the whole family rather than only the two new names.

## Specification text

The prose below is what a release folds into the next version document, at the anchors
given. It was written against `versions/1.2.md` and is carried here unchanged.

**Anchor:** Declarative glue > The notify block — and `attach: print`, sending the document itself

#### Links back to the application: `{recordUrl}`, `{inboxUrl}`, `{appUrl}`

A notification that cannot be acted on is a notification that gets ignored. "You have an approval
waiting" is only useful if it carries the way back to the record, so `subject` and `body` accept three
reserved link placeholders alongside the field ones:

| Placeholder | Resolves to |
| --- | --- |
| `{recordUrl}` | the record the message is about, opened in the application |
| `{inboxUrl}` | the recipient's task inbox |
| `{appUrl}` | the application's external base URL - the origin only |

```yaml
    notify:
      to: Approver.email
      subject: "Approval needed: invoice {number}"
      body: "Open it here: {recordUrl}\nEverything waiting on you: {inboxUrl}"
```

> **Normative.** The three names are RESERVED at every notify call site: an entity field of the same
> name MUST NOT shadow them. `{recordUrl}` and `{inboxUrl}` MUST be resolved by the implementation to
> a complete address; an intent MUST NOT be required to spell a route, and a conforming implementation
> MUST NOT require one. `{appUrl}` yields the origin alone - it is the escape hatch for addresses the
> other two cannot express, and everything appended to it is authored text.

Why the intent never writes the path: the routes belong to whatever renders the application, and an
intent that named one would encode a layout it does not own - correct only until that layout changes,
and silently wrong afterwards. `{recordUrl}` states the destination; the implementation states the
address.

> **Normative.** Inside a [`forEach`](#one-message-per-related-row-foreach) fan-out `{recordUrl}` links
> the ROW, like every other bare path in the block - the row is what that message is about. The anchor
> record is addressable for VALUES only.

**Anchor:** Appendix A: DSL index

| [notify link placeholders](#links-back-to-the-application-recordurl-inboxurl-appurl) | `{recordUrl}` / `{inboxUrl}` / `{appUrl}` - a message that carries the way back into the application |
