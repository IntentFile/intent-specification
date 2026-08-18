# A declared payload for outward-facing messages


- **Status:** released in [1.5](../versions/1.5.md)
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/6768
- **Discussion:** https://github.com/IntentFile/intent-specification/pull/31

## Why

An `integrations` entry can only send **the record as stored**. That is only right when the receiver accepts the entity, and it costs something even then: every column becomes part of a public contract, so adding a field silently changes what the outside world receives.

A real integration contract is usually an *envelope* — a type, a version, an idempotency key, a timestamp, an identifier of the sender — which no arrangement of entity columns can produce. Today an application honouring such a contract writes the sender by hand, which also means the contract is invisible to the model.

## What this adds

`payload` on an outward-facing entry, declaring that envelope key by key:

```yaml
payload:
  type: "user.assignment.requested"     # literal
  version: 1
  messageId: "{uuid}"                   # minted per message
  tenantId: "{tenant}"                  # execution context
  appId: "@config:APP_ID"               # configuration
  email: email                          # a field of the record
  role: role.name                       # one hop off a to-one relation
  requestedAt: "{now}"
```

The value forms are the ones `notify` already resolves — deliberately borrowed, not invented — plus `@config:KEY` and a **closed set of four** context tokens: `{uuid}`, `{now}`, `{tenant}`, `{user}`.

## The normative half

The cap is what makes this a contract rather than a transformation language, so what falls outside it is an authoring error, not a run-time surprise: interpolated text, a nested object or list, a multi-hop path, an unknown token, and a payload on a method that carries no request body. Keys are sent in the order they were declared. A bare word naming nothing on the record is a literal (the only way to carry a one-word constant); a braced value is a reference and must resolve.

Appendix A gains its row.

## Notes

The same vocabulary is meant to serve the emitting construct and the arrival mapping when those are specified — designed once here rather than twice later.

Proven out by the reference implementation in eclipse-dirigible/dirigible#6781 (parse-time refusals, the generated sender, and an end-to-end assertion that the declared payload — not the record — is what goes on the wire).

## Specification text

The prose below is what a release folds into the next version document, at the anchors
given. It was written against `versions/1.2.md` and is carried here unchanged.

**Anchor:** Declarative glue > integrations — outbound HTTP

#### payload — the declared envelope

Without a `payload`, the request body is the record as stored. That is only right when the receiver accepts the entity, and it has a cost even then: every column becomes part of a public contract, so adding a field silently changes what the outside world receives. A real integration contract is usually an *envelope* — a type, a version, an idempotency key, a timestamp, an identifier of the sender — which no arrangement of entity columns can produce.

`payload` declares that envelope, key by key:

```yaml
integrations:
  - name: requestUserAssignment
    event: { onCreate: UserInvitation }
    method: POST
    url: "@config:ASSIGNMENT_URL"
    payload:
      type: "user.assignment.requested"     # literal
      version: 1
      messageId: "{uuid}"                   # minted per message
      tenantId: "{tenant}"                  # execution context
      appId: "@config:APP_ID"               # configuration
      email: email                          # a field of the record
      role: role.name                       # one hop off a to-one relation
      requestedAt: "{now}"
```

The value forms are the ones [`notify`](#notifications) already resolves, deliberately borrowed rather than invented: a **literal**, a **direct field**, or a **one-hop `relation.field`** of a to-one relation, which the generated sender reads from the related record it loads once. `@config:KEY` reads the configuration, as it does in `url`.

The **context tokens** are a closed set of four:

| token | value |
|---|---|
| `{uuid}` | a fresh identifier, minted per message — the idempotency key a receiver deduplicates on |
| `{now}` | the send time, as an ISO-8601 instant |
| `{tenant}` | the tenant the send runs for |
| `{user}` | the user behind the change that raised the event |

> **Normative.**
> A `payload` value MUST be one whole value in one of the declared forms. Interpolated text (`"Order {id} placed"`), a nested object and a list are NOT payload values and MUST be reported as authoring errors — a payload is a contract, not a template.
> A path MUST resolve at most one hop; `a.b.c` MUST be rejected.
> An unknown context token MUST be an authoring error, never an empty value in a sent message.
> A `payload` MUST be rejected on a method that carries no request body.
> Keys MUST be sent in the order they were declared.
> A bare word that names no field and no to-one relation of the record is a **literal** — the only way to carry a one-word constant. A value braced as `"{name}"` is a reference and MUST resolve.

Three value forms and four tokens is the cap, and the cap is the point: it expresses a frozen contract without the construct becoming a transformation language. A payload that needs more than this is an algorithm, and belongs in a hand-written handler — the honest hand-off named in [the scope boundary](#the-scope-boundary).

**Anchor:** Appendix A: DSL index

| [`integrations.payload`](#payload--the-declared-envelope) | the declared envelope a message carries, instead of the record as stored |
