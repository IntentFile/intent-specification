# A grant that adds its role to a gate and keeps the roles the gate already holds

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8852
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8853
- **Builds on:** [0127 - what a deployment's shared roles may not reach](0127-permissions-exclusive-extra-roles-separate-from.md), [0152 - a role the file authors is a role a person can be given](0152-authored-roles-assignable.md)

## The problem

A resource that a `permissions` entry names is gated by the roles the file authors for it. The
roles a generator derives for a resource that no grant names - one read and one write role per
entity, one read role per report - are then no longer its gate.

That is right when the file takes a resource over. It is wrong for the commonest small case: an
application adds ONE role for its people to a reference entity, and the derived roles of that
entity must keep working, because machine clients hold them. The file has to author the derived
role names itself, next to the role it wanted to add:

```yaml
permissions:
  - { role: vacations.LeaveType.LeaveTypeReadOnly,   can: [LeaveType:read] }
  - { role: vacations.LeaveType.LeaveTypeFullAccess, can: [LeaveType:write] }
  - { role: Employee, description: Self-service, can: [LeaveType:read] }
```

The gates come out right, but two generated names are now roles the file authors, and by 0152 every
authored role is a role a person may be given. An application with one role for people and four
reference entities offers the administrator of an organisation eleven roles, eight of which mean
nothing to them. It also ties the file to a naming convention that belongs to the generator.

## The proposed shape

```yaml
permissions:
  - { role: Employee, description: Self-service, can: [LeaveType:read, PublicHoliday:read], additive: true }
```

## Expected behaviour

- **The role is added, nothing is replaced.** Every resource the entry binds keeps the gate it has
  without the entry, and the entry's role is added to it: to the read gate for a read action, to
  the write gate and the read gate for a write action or `*`.
- **A resource only additive entries name keeps its derived roles.** They stay its gate, stay
  declared, and stay unmarked: by 0152 they are not offered to a person. The additive role is a
  role the file authors, so it is offered. In the example a holder of the derived read role still
  reads `LeaveType`, a holder of the derived write role still writes it, a holder of `Employee`
  reads it and is refused a write.
- **A resource another entry also names** is gated by that entry's roles, as it is today, and by
  the additive role beside them. The order of the entries does not matter.
- **A resource an `exclusive` entry of another role names** (0127) stays closed to the roles a
  deployment adds to every gate, and the additive role is added to it.
- **A composition child with no grants of its own** follows its master's gate, the added roles
  included. An additive entry that names the child itself adds to the gate the child follows; it
  does not detach the child from its master.
- **A report** follows the same rule: it keeps its derived read role beside the added one.
- **A resource a generator derives no role for** (an export) has nothing to keep: the roles of the
  entries naming it are its gate, additive or not.
- **Every surface that reads the gate admits the added role**: the entity's own endpoints, the
  create-from, transition and print endpoints of 0120, and a file import into its rows.

## Edge rules

- `additive: true` and `exclusive: true` on the same entry is an authoring error: the one keeps the
  roles a gate holds and the other narrows the gate to the roles the file names. The two may be
  declared on separate entries, of the same role or of different ones.
- An additive entry never removes a role from a gate. Whether a resource keeps its derived roles,
  and whether it takes the roles a deployment adds to every gate, is decided by the other entries
  alone.
- If no other role may write a resource that non-additive entries name, an additive write grant
  makes its role the one that may.
- A token whose action maps to no read or write gate (`Order:approve`) is unaffected by the key.
- The default is `false`. A file that does not declare the key regenerates unchanged.

## Prior art / workarounds

Authoring the derived role names by hand, as shown above, which turns them into roles offered to
people. A flag keeping one authored role out of that offer was the alternative considered; it
treats the symptom, and the file would still restate the generator's naming convention. Reference
implementation: https://github.com/eclipse-dirigible/dirigible/pull/8853.

## Specification text

**Anchor:** Scoped surfaces & roles > permissions, after 0127's paragraph and normative block for
`exclusive`.

A resource a grant names is gated by the roles the file authors for it, in place of the roles a
generator derives for a resource that no grant names. A grant that only adds a role, and leaves the
gate otherwise as it is, is declared `additive`:

```yaml
permissions:
  - { role: Employee, can: [LeaveType:read], additive: true }
```

`LeaveType` is then read by its derived read and write roles and by `Employee`, and written by its
derived write role. Only `Employee` is a role the file authors.

> **Normative.**
> A `permissions` entry declaring `additive: true` MUST add its role to the gates of every resource
> it binds - the read gate for a read action, the write gate and the read gate for a write action -
> and MUST NOT remove a role from them. A resource that only additive entries bind MUST keep the
> roles a generator derives for it as its gate, declared and not marked as roles a person may be
> given. A resource that another entry also binds is gated by that entry's roles and the additive
> role; a resource an `exclusive` entry binds stays closed to the roles a deployment adds to every
> gate. A composition child with no grants of its own follows its master's gate, the added roles
> included, and an additive entry binding the child itself adds to that gate. Every generated
> surface that reads the resource's gate MUST admit the added role. An entry declaring both
> `additive: true` and `exclusive: true` MUST be reported as an authoring error.

## DSL index

| Construct | What it does |
| --- | --- |
| [`permissions[].additive`](#permissions) | `true` adds the entry's role to the gates of the resources it binds and keeps the roles those gates already hold, the derived ones included |
