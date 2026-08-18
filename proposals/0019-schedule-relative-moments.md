# A schedule's `where` value may be a moment relative to now

- **Status:** released in [1.5](../versions/1.5.md)
- **Issue:** [IntentFile/intent-specification#28](https://github.com/IntentFile/intent-specification/issues/28)

## The problem

The archetypal schedule is a **staleness sweep**: "rows still provisioning after 30 minutes",
"quotations unanswered for 7 days", "carts abandoned for an hour". `schedules:` exists for exactly
this shape — run on a cron, query an entity, act per matching row — and it can express every part of
such a sweep except the one that makes it a sweep: *how old is too old*.

A `where` value is a literal or the current moment. There is no way to write a moment **relative** to
now, so the closest expressible query is

```yaml
- { field: modifiedAt, op: lt, value: CURRENT_TIMESTAMP }
```

which matches every row ever modified. Two workarounds remain, and both are worse than the construct:

- **Store the deadline as a column** and have every writer keep it current — modelling the clock into
  the data so that the comparison can be against a literal.
- **Drop out of `schedules:`** into a hand-written scheduled job that runs the very query the format
  can already describe and then calls the very notification it would already have generated. This is
  what happens in practice, and it is a hand-off that buys nothing: both halves were inside the
  boundary; only a value form was missing.

Relative moments are already ordinary one construct over — a report filter compares against the
current date. The schedule's `where` is the one place a time *window* cannot be stated.

## The proposed shape

An offset on the existing moment tokens, in the existing value slot. No new operator, no new key.

```yaml
schedules:
  - name: stuckProvisioning
    cron: "0 */5 * * * ?"
    entity: TenantApplication
    where:
      - { field: provisioningStatus, op: eq, value: Provisioning }
      - { field: changedAt,          op: lt, value: "CURRENT_TIMESTAMP-PT30M" }
    notify:
      to: ops@example.com
      subject: "Tenant application {id} has been provisioning for over 30 minutes"
      body: "It may need an operator."

  - name: unansweredQuotations
    cron: "0 0 8 * * ?"
    entity: Quotation
    where:
      - { field: status, op: eq, value: Sent }
      - { field: sentOn, op: lt, value: "CURRENT_DATE-P7D" }
    notify: { to: owner.email, subject: "Quotation {id} has had no answer for a week" }
```

## Expected behaviour

- A value of the form `<moment token><+|-><ISO-8601 duration>` resolves, at **each firing**, against
  that run's clock — `CURRENT_DATE-P7D` is seven days before today, `CURRENT_TIMESTAMP-PT30M` is
  thirty minutes ago. A conforming generator MUST NOT bake the resolved instant into the generated
  artifact: a sweep generated on Monday must not still be asking about Monday.
- The forward form (`+`) is admitted symmetrically, for "falls due within the next week".
- The comparison happens **in the queried field's own shape** — a date field against a date, a
  timestamp field against an instant — the rule the format already follows for a `now` default.
- This is a **moment vocabulary, not an expression language**: exactly one offset on one token. No
  arithmetic between fields, no nesting, no further operators.
- Existing files are unaffected: today's values keep their meaning exactly.

## Edge rules

Each of the following MUST be an authoring error, reported at generation, rather than a comparison
that silently never matches — which is the failure mode this whole construct exists to remove:

- **An offset the shape cannot carry.** A date has no time component, so a time offset on a date
  (`CURRENT_DATE-PT30M`) is an error rather than an amount quietly rounded to a day.
- **An offset that is not a single ISO-8601 duration** — a bare amount (`-30M`), or a second offset
  (`CURRENT_TIMESTAMP-P7D-P1D`).
- **A moment compared with a non-temporal field.**
- **A token of the other shape than the field's.** A timestamp moment handed to a date column is the
  same defect whether or not it carries an offset, so this rule applies to a bare token too; naming it
  only for offsets would make the shape rule an accident of syntax.

A field the queried entity does not itself declare — a generated audit column, or a field of a source
owned by another model — is exempt from the shape and temporality checks, because the properties are
not resolvable at that point. The token's own shape then decides. This matters: an audit column is
where a staleness sweep most often looks.

## Prior art / workarounds

The two named above (a maintained deadline column; a hand-written job re-implementing the query and
the action). Both are visible in real applications, and both hide from the model something the model
is otherwise complete about: *this application sweeps for stale rows, on this window*.
