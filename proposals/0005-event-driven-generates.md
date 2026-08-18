# Event-driven creation: a `generates` action may run on a source event


- **Status:** released in [1.3](../versions/1.3.md)
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/6711
- **Discussion:** https://github.com/IntentFile/intent-specification/pull/14

## The gap

`generates:` (create-from) was strictly a **user action**: a button on the source view. There was no way to say "when the source reaches this state, create the follow-up document".

The shape that keeps coming up: a record is completed by an earlier automated or human step, and a document must be minted from it **without another click**. A fine arrives by webhook; once the responsible person is identified on it (a status transition), a declaration document must be created from the fine and that person. The expressible options were all wrong:

- a `generates` button plus a `wait` step — the automation degrades to a person remembering to click, and an unclicked record parks its process instance indefinitely;
- `posts` — event-driven and idempotent, but it emits **flat mapped rows** and cannot mint a header-and-items *document* (the item rows need the id of the header it would have to create first);
- hand-written code.

The 1.1 "Planned" list named this gap; this PR closes it and removes the entry.

## What it adds

`versions/1.2.md` gains **`#### Event-driven creation — `event:`** under *generates — create-from*, plus an *Appendix A* row and a "what this version adds" bullet.

```yaml
generates:
  - name: declaration-from-fine
    from: Fine
    to: Declaration
    event: { onTransition: Fine, when: "Status == IDENTIFIED" }   # or { onCreate: Fine }
    map:
      Fine: id                       # REQUIRED with an event — the back-reference, i.e. the guard
      Vehicle: Vehicle
    defaults: { declaredAt: now }
    items:
      - { name: "Fine {number}", amount: Amount }
```

Three normative rules, each answering a way this could go quietly wrong:

- **The event says WHEN, never what.** It names the entity `from:` already declares, and never repeats the owning model (`fromUses:` owns that). Two ways to name one source could only drift. The guard is evaluated against the source **re-read at delivery**, never the payload — which is as-of the event and lacks anything a later step wrote.
- **At most once.** `map` must copy the source's primary key onto the target's to-one back to the source, and the creation must return the target that already back-references it. Without that back-reference a redelivery mints a duplicate document, so a declaration lacking it is rejected. A create-from with *no* event keeps no such guard — producing several targets from one source by clicking twice is a legitimate manual act.
- **Declaring an event drops the button** unless `button: true` asks for both; `button: false` with no event is rejected (the action would have no trigger at all). When both triggers exist they must share one creation path, and therefore one guard.

## Reference implementation

Eclipse Dirigible: eclipse-dirigible/dirigible#6711 — the listener is generated as a message handler on the source's `-transitioned` (or create) topic that re-reads the source, applies the guard and calls the same create-from a button would, carrying no mapping of its own. Verified at the outermost layer: posting the source mints the whole document, line items included, with nobody calling the create-from, and a click afterwards returns that same document.

## Specification text

The prose below is what a release folds into the next version document, at the anchors
given. It was written against `versions/1.2.md` and is carried here unchanged.

**Anchor:** The Intent File Specification > Version 1.2

- **[Event-driven creation](#event-driven-creation--event)** — a create-from may declare an `event:` and mint the whole document (header and items) when the source reaches a state, at most once, instead of waiting for someone to press a button.

**Anchor:** Declarative glue > generates — create-from > Prompted input — `prompt:`

#### Event-driven creation — `event:`

A create-from MAY declare an `event:` instead of relying on the button — the follow-up document is minted the moment the source reaches a state, with nobody clicking. The canonical case is a document that arrives from the outside and is completed by an earlier step: a fine ingested by a webhook, whose responsible person is identified by a transition, must produce a declaration document from the fine and that person.

```yaml
generates:
  - name: declaration-from-fine
    from: Fine
    to: Declaration
    event: { onTransition: Fine, when: "Status == IDENTIFIED" }   # or { onCreate: Fine }
    map:
      Fine: id                       # REQUIRED with an event — the back-reference, i.e. the guard
      Vehicle: Vehicle
    defaults: { declaredAt: now }
    items:                           # a whole document — header AND items
      - { name: "Fine {number}", amount: Amount }
```

**Normative.** Exactly one trigger MUST be declared: `onTransition` (a status write — a `when: "<StatusRelation> == <status>"` guard is mandatory, the status named or numbered) or `onCreate` (the source's insert — the guard is optional, for a source with no status lifecycle). The entity named there MUST be the entity `from:` declares; the owning model is never repeated (`fromUses:` declares it). The guard MUST be evaluated against the source as re-read at delivery, not against the event payload, which is as-of the event and lacks anything a later step wrote.

**Normative.** An event-driven create-from is **at-most-once**: `map` MUST copy the source's primary key onto a to-one relation of the target back to the source, and the generated creation MUST return the already existing target instead of creating a second one. A file declaring an `event` without that back-reference MUST be rejected — a redelivery would otherwise mint a duplicate document. A create-from with no `event` carries no such guard: producing several targets from one source by clicking twice is a legitimate manual act.

**Normative.** Declaring an `event` drops the button unless `button: true` is declared as well; `button: false` without an `event` MUST be rejected (the action would have no trigger at all). When both triggers are declared they MUST share one creation path, and therefore one at-most-once guard.

`sourceStatus:` composes unchanged: the flip happens once the target exists, and cannot re-trigger the create-from because the guard has already claimed the source.

Prefer this over [`posts`](#posts--derived-rows-on-an-event) when the result is a document with line items — `posts` emits flat mapped rows and cannot reference the freshly created header. Prefer it over a button plus a [`wait`](#wait--park-the-process-on-a-data-event) step when the step is really waiting for a person to remember to click: an unclicked record parks its process instance indefinitely.

**Anchor:** Appendix A: DSL index

| [`generates.event`](#event-driven-creation--event) | mint the document on a source event instead of a click, at most once |

**Anchor:** Appendix A: DSL index > Planned — recognised but not yet implemented

- A declarative state machine, and shadow audit-history entities (audit *columns* via `audit: true` ship today).
