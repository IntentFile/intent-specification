# `lifecycle` - the legal status graph, enforced on every status write


- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/6714
- **Discussion:** https://github.com/IntentFile/intent-specification/pull/16

## The gap

Everything else about statuses is stated one edge at a time: `init:` names where a record starts, a `transitions` button guards the flips a user performs through it, a workflow step sets one, a check files a rejected record in another. Nowhere does a conforming file say which moves are legal **at all** - so any writer that is not a transition button (a workflow branch, a glue action, an API call) can move a document from any status to any other, and nothing notices.

The 1.1 and 1.2 Planned lists named this ("a declarative state machine").

## The construct

```yaml
- name: SalesInvoice
  lifecycle:
    edges:
      - { from: DRAFT,  to: [ISSUED, CANCELLED] }
      - { from: ISSUED, to: [PAID, VOIDED] }
```

One entry per source status; either side accepts a seeded status name or its id. The graph is always over the entity's `function: EntityStatus` relation, so it names no column, and the nomenclature must be seeded in the same file (a status entity owned by another model is seeded there, and so is its lifecycle).

## Two normative rules

1. **Every status write is validated against the graph** - user, workflow, glue, transition button alike - so enforcement belongs to the layer every writer passes through, never to the transition endpoints alone, which would leave every other writer unguarded. Where the status relation declares `init:`, a record must also be *created* at the start: entering the lifecycle anywhere else skips the graph rather than travelling it.
2. **`transitions` become presentation over the edges**: each `from` must reach its `setStatus` along a declared edge, and a status written by a workflow step or forced by a check's rejection must be one some edge reaches - reported when the file is read. A reject path transiting through an approved status is exactly the mistake the graph exists to catch.

## Also in this PR

- The construct joins the "Status references - name, not number" site list and *Appendix A: DSL index*.
- `transitions` gains a sentence pointing at the graph.
- "a declarative state machine" leaves the Planned list.

Implemented in Eclipse Dirigible - eclipse-dirigible/dirigible#6714 (PR eclipse-dirigible/dirigible#6733), which is where the wording above was proven out (including the `on:` key the shape deliberately does not have: it would be redundant, and YAML 1.1 reads a bare `on` as the boolean `true`).

## Specification text

The prose below is what a release folds into the next version document, at the anchors
given. It was written against `versions/1.2.md` and is carried here unchanged.

**Anchor:** stamped at a modeled issue step (a placeholder holds the field until then): > immutableWhen / immutable — user-write immutability


### lifecycle — the legal status graph

Everything else about statuses is stated one edge at a time: `init:` names where a record starts, a [`transitions`](#transitions--guarded-status-flips) button guards the flips a user performs through it, a workflow step sets one, a [check](#checks--declarative-validations) files a rejected record in another. Nowhere does the file say which moves are legal *at all* — so any writer that is not a transition button (a workflow branch, an API call, a custom action) can move a document from any status to any other, and nothing notices.

`lifecycle:` states the whole graph, once:

```yaml
- name: SalesInvoice
  lifecycle:
    edges:
      - { from: DRAFT,  to: [ISSUED, CANCELLED] }
      - { from: ISSUED, to: [PAID, VOIDED] }
```

- One entry per **source** status, listing every status reachable from it. Both sides accept a [seeded status name or its id](#status-references--name-not-number).
- The graph is always over the entity's `function: EntityStatus` relation, so it names no column; the nomenclature MUST be seeded in the same file (a status entity owned by another model is seeded there, and so is its lifecycle).
- A status not listed as any `from` is **terminal**; a status listed nowhere is simply unreachable through this entity.

> **Normative.** A conforming generator MUST validate every status write against the graph — user, workflow, glue, transition button alike — and reject a move no edge declares, with a message naming both statuses. Enforcement therefore belongs to the layer every writer passes through (the generated persistence layer), never to the transition endpoints alone, which would leave every other writer unguarded. Where the status relation declares `init:`, a record MUST also be *created* in that status: entering the lifecycle anywhere else skips the graph rather than travelling it.

> **Normative.** With a lifecycle declared, `transitions` become **presentation over its edges**: each `from` status of a transition MUST reach its `setStatus` along a declared edge, and a status written by a workflow step or forced by a check's rejection MUST be one that some edge reaches. A conforming generator reports the disagreement when the file is read, not when the button is pressed — a reject path transiting through an approved status is exactly the mistake the graph exists to catch.

It composes with the [`stage:` classification](#stage--what-a-status-means-to-the-lifecycle): a stage says what a status *means* (draft, live, cancelled, void) and scopes reports by it; the lifecycle says how a record may *move* between statuses.

**Anchor:** Declarative glue > transitions — guarded status flips

When the entity declares a [`lifecycle`](#lifecycle--the-legal-status-graph), a transition is presentation over its edges: its `from`/`setStatus` pair must be one, and the graph — not the button — is what every other writer is held to as well.

**Anchor:** Data, seeds & naming > seeds > Status references — name, not number

Everywhere the file names a status — a [transition's](#transitions--guarded-status-flips) `from` and `setStatus`, a relation's `init`, a status-setting step's `value`, [`abortOn`](#aborton--cancel-the-instance-on-a-terminal-status)'s `status`, a [check's](#checks--declarative-validations) `status` / `setStatus`, [`immutableWhen`](#immutablewhen--immutable--user-write-immutability), a [`lifecycle`](#lifecycle--the-legal-status-graph) edge, a [posting's](#postings--source-document-to-ledger) event guard, a [report's](#reports) `filter` — the seeded **name** may be written instead of the id:

**Anchor:** Appendix A: DSL index

| [`lifecycle`](#lifecycle--the-legal-status-graph) | the whole legal status graph, enforced on every status write |

**Anchor:** Appendix A: DSL index > Planned — recognised but not yet implemented

- Event-driven **document** generation (produce a whole document, rather than the mapped rows of [`posts`](#posts--derived-rows-on-an-event), on an event), and shadow audit-history entities (audit *columns* via `audit: true` ship today).
