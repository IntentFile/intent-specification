# A dropped file lands in one tenant - the inbound folder is per tenant

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8157
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8810

## The problem

An [`inbound`](#inbound--arrivals-from-outside) `folder` source is polled on its cron. On a shared
deployment the poll runs once per tenant, and every tenant reads the same declared folder: a file
dropped there is ingested by whichever tenant's tick reaches it first - or by several, if their
ticks overlap before the file is moved - so a supplier's price list lands in a customer it was never
meant for. The record is in the wrong tenant's database, and nothing about the drop said which
tenant it was for.

## The proposed shape

No new key. The declared folder stays what the author wrote.

```yaml
inbound:
  - { name: leadDrop, source: { folder: /data/inbox/leads, cron: "0 */5 * * * ?" }, create: Lead }
```

## Expected behaviour

- Each tenant reads only its own folder. The default tenant of a deployment reads the declared
  folder; every other tenant reads `<folder>/tenants/<tenant id>`.
- A file dropped into a tenant's folder is ingested into that tenant only, and moved out of that
  folder like any other read file.
- A single-tenant deployment reads the declared folder, as before.

## Edge rules

- A tenant id that cannot be used as one folder name reads nothing, and the implementation reports
  it.
- The webhook and the queue / topic arrivals are not concerned: they already arrive in one tenant.

## Prior art / workarounds

One declared folder per tenant, which a model cannot know.

## Specification text

**Anchor:** Declarative glue > `inbound — arrivals from outside`, after its normative block.

On a deployment shared by tenants, a dropped file belongs to one of them, and the folder says which.

> **Normative.**
> Where a deployment serves several tenants, a `folder` source MUST be read once per tenant, and each
> tenant MUST read only its own folder: the deployment's default tenant the declared folder, every
> other tenant the folder `tenants/<tenant id>` under it. A file MUST be ingested into the tenant whose
> folder it was read from, and into no other. A tenant whose id cannot name a single folder MUST read
> nothing, and the implementation MUST report it.

## DSL index

No new row; the `inbound` row is unchanged.
