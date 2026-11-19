# A role the file authors is a role a person can be given

- **Status:** released in [1.13](../versions/1.13.md)
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8728
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8731

## The problem

`permissions:` generates a deduplicated set of roles, and a generator that materialises the
application's surfaces derives more roles of its own: one read and one write role per entity when
the file grants nothing for it. Nothing tells the two kinds apart. A deployment that lets the
administrator of an organisation give its people roles therefore has two choices: offer every role
the application declares, which for an application with fifty entities is a list of a hundred
generated names nobody can choose from, or offer a fixed pair of broad roles and nothing else. In
the second case every person can do everything, or nothing: the clerk who enters invoices, the
reader of the stock and the person who requests leave all hold the same role, although the file
already names the narrower roles they need.

## The proposed shape

No new key. The roles a file already authors are the ones meant for people:

```yaml
permissions:
  - { role: Accountant, description: Enters and issues invoices, can: [Invoice:write, Customer:read] }
  - { role: Warehouse,  description: Reads the stock,            can: [StockItem:read] }
```

## Expected behaviour

- Every role a `permissions:` entry authors is declared as a role a person may be given, together
  with its `description`.
- The roles a generator derives for a resource no grant names (one read and one write role per
  entity) are not marked so. They stay declared and usable, by a machine client or by an
  administrator granting them by hand, but they are not offered as a person's role.
- A deployment that manages the roles of its people offers exactly the marked roles, plus whatever
  roles of its own it always offers, and refuses any other. An application whose file authors no
  role offers only the deployment's own.

## Edge rules

- A role is offered once, whichever file or module declares it.
- The mark adds nothing to what the role may reach: its gates are the grants of its `can:` tokens,
  as before.
- A deployment's own roles (an operator's or an administrator's of the deployment itself) are never
  offered to a person through the application, even when a file authors a role of the same name.

## Prior art / workarounds

Before this, a deployment offered a fixed pair of broad roles to the people of an organisation, and a
narrower role could only be granted by hand outside the application, where the application's list of
members did not show it. Reference implementation: https://github.com/eclipse-dirigible/dirigible/pull/8731.

## Specification text

**Anchor:** Surfaces > permissions, appended after the paragraph that follows the example.

The roles a file authors are the roles its people are given. The roles a downstream generator
derives for a resource that no grant names are not.

> **Normative.**
> A conforming generator MUST declare every role a `permissions:` entry authors as one a person may
> be given, with its `description`, and MUST NOT mark so a role it derives for a resource that no
> grant names. A deployment that manages the roles of its people MUST offer the marked roles, MAY add
> roles of its own, and MUST refuse any other role it is asked to give. The mark grants nothing: what
> a role may reach is still decided by its `can:` tokens. A role of the deployment itself is never
> offered to a person through the application.

## DSL index

| Construct | What it does |
| --- | --- |
| [`permissions`](#permissions) | roles - the ones a person may be given |
