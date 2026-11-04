# A posting refused by a closed period is deferred, not lost, and is posted again when the period reopens

- **Status:** draft
- **Issue:** https://github.com/IntentFile/intent-specification/issues/163, https://github.com/eclipse-dirigible/dirigible/issues/7703
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/7709
- **Amends:** [0028 - period locking](0028-period-locking.md)

## The problem

A posting's target can carry a period lock: the journal entry declares
`immutableInPeriod: { period: AccountingPeriod, date: entryDate }`. 1.11 states what the lock does
to a **user** write and says that "writes performed by the application itself - a workflow step, a
roll-up, a generated reversal - stay possible". It says nothing about a posting whose target would
fall in a closed period.

The scenario is ordinary: an invoice dated in March is issued in April, after March was closed. The
posting runs after the invoice's own transition has committed, so the invoice is issued, numbered
and answered as a success, while the entry it should produce is refused by the lock. Read one way
the refusal is lost in a log and the invoice stays unposted for good - reopening March is a write on
the register, which the posting does not listen to. Read the other way the posting books into a
closed period, which is what closing it was meant to prevent.

## The proposed shape

No new key. The behaviour applies to every posting whose `creates` target declares
`immutableInPeriod`.

```yaml
entities:
  - name: JournalEntry
    immutableInPeriod: { period: AccountingPeriod, date: entryDate }

postings:
  - name: salesInvoicePosting
    event: { onTransition: SalesInvoice, when: "Status == ISSUED" }
    creates: JournalEntry
    backReference: SalesInvoice
    map: { entryDate: date }            # the target's locked date, copied from the source
```

## Expected behaviour

- **Deferred.** Before it writes anything, the posting asks the register the question the target's
  lock would ask. A target date in a closed period writes nothing; the refusal is recorded, naming
  the date and the source document, and the source stays unposted. The source's own transition is
  not refused: the source is routinely another model's, which has no register to consult.
- **Posted again.** When the period is reopened - by a transition or by a plain edit of the register
  row - every source the posting's own guard selects, dated inside the reopened period and not yet
  back-referenced by a target, is posted again. A register write that leaves the period closed posts
  nothing.
- **Through the posting's own body.** The re-run applies the status guard, the determination rule,
  the at-most-once back-reference and the period check exactly as the event does; it only decides
  which sources are offered again, and never posts one the posting itself would not.
- A posting whose target carries no period lock is unaffected and generates as before.

## Edge rules

- The register is an entity of the target's model (0028), so everything the check and the re-run
  need is local to the posting's target model.
- A redelivery against a period that stays closed for days is not a retry loop: the deferred source
  waits for the reopen, not for the next delivery.

## Prior art / workarounds

None within the file. The refusal reached only the posting's error log, the issuer was answered as
a success, and the document stayed unposted for good, since reopening the period was a write the
posting did not listen to. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/7709.

## Specification text

**Anchor:** Entities & fields > immutableInPeriod / period - period locking, appended after its
**Normative** block and before "lifecycle - the legal status graph".

A [posting](#postings--source-document-to-ledger) whose target is guarded is the application writing,
but a posting into a closed period is exactly the figure the close was meant to stop moving. So it
does not write: it is **deferred**. The source document has committed by then - it is routinely
another model's, which has no register to ask - and stays as it is, unposted, with the refusal
recorded against it. Reopening the period posts it.

> **Normative.** A posting whose target would fall in a closed period MUST NOT write; the refusal is
> recorded against the source document, and the source stays unposted. When the period is reopened,
> by a transition or a plain edit of the period register, every source the posting's own guard
> selects, dated inside the reopened period and not yet back-referenced by a target, MUST be posted
> again through the posting's own body - its guard, its determination rule, its at-most-once
> back-reference and this period check apply as on the event. A register write that leaves the
> period closed MUST post nothing. A target without a period lock is unaffected.

<!-- editor: "dated inside the reopened period" reads the source value the target's locked date is
     copied from by the posting's `map`. The implementation identifies that source value where the
     `map` cell is a plain copy; it is silent on a computed date cell, and so is this text. -->
