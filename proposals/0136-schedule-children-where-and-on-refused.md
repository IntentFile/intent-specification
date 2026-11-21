# A scheduled generation's children take a row rule, and a child the target refuses can be left out

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8491
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8531

## The problem

A scheduled `generate` may fan out `children` over the rows of a collection
(`forEach: { entity, match }`). Two things cannot be said about them.

1. **Which rows qualify.** Every row the `match` finds becomes a child. A monthly project run creates
   one timesheet per assignment of the project, and an assignment of an employee who has left gets a
   timesheet like any other.
2. **What one refused child costs.** A source row's whole generation is one unit of work, so a child
   the target's own rules refuse rolls back the header and every sibling. A monthly run that books
   eight hours per working day loses the whole project's month to one day the target refuses (before a
   hire date, outside the project's dates, over 24 hours on one calendar day). Not refusing those days
   books them.

## The proposed shape

```yaml
schedules:
  - name: monthly-project-timesheets
    cron: "0 0 2 1 * ?"
    entity: Project
    model: projects                  # the projects and their staffing belong to another model
    generate:
      to: ProjectTimesheet
      unique: [Project, period]
      map: { Project: id }
      defaults: { period: now }
      children:
        - to: EmployeeTimesheet
          parent: ProjectTimesheet
          forEach: { entity: EmployeeProjectAssignment, model: projects, match: { Project: id } }
          where:
            - { field: Employee.active, op: eq, value: true }   # the employee belongs to a third model
          map: { Employee: Employee }
          onRefused: skip
          children:
            - to: EmployeeDayAllocation
              parent: EmployeeTimesheet
              forEach: { days: workingDays }
              dayField: day
              defaults: { hours: 8 }
              onRefused: skip
```

## Expected behaviour

- `where:` on an `entity` child is the list of `{ field, op, value }` conditions a `schedules[].where`
  carries, with the same operators and the same moment values. The conditions are ANDed. A `field` is a
  field or to-one relation of the collection entity, or one relation hop, `<relation>.<field>`, read on
  the row that relation points at. Only the rows that satisfy every condition become children. A row
  whose relation is empty never satisfies a hop condition.
- The collection may be owned by another model (`forEach.model:`), and a hop may lead into the model
  that owns the related entity - a third one, when the collection's owner itself refers to it there
  (the example's assignment belongs to the projects model, its employee to the employees model). The
  fields are those of the owning models.
- A condition on the collection's own status, or another seeded relation, may name the seed, as in a
  schedule's `where`, when the collection belongs to this model.
- `onRefused: skip` on a children entry: a child that the target refuses through its own rules is
  left out, together with everything written for it beneath it. The refusal is logged with the
  schedule, the child, the key of the collection row (or the day) and the key of the source row. The
  header and the other children are still created. Any other failure still fails the whole source row,
  as before.
- The writes of a skipped child, and the announcements of those writes, are taken back. Its own
  children go with it, unless their entry declares `onRefused: skip` itself; then each of them is
  judged on its own.
- A header's totals, counts and roll-ups reflect only the children that were created.

## Edge rules

Refused when the intent is read:

- `where:` on a `days` child (there is no row to filter);
- a condition with no field, an operator outside `eq` / `ne` / `gt` / `ge` / `lt` / `le` / `like`, a
  moment of the wrong shape for the field it is compared with;
- a field the collection entity does not have, a hop through a name that is not a to-one relation of
  it, a hop to a field the related entity does not have, and a path longer than one hop. For an entity
  another model owns, these are checked against that model when the model is generated;
- an `onRefused` value other than `skip`.

A children entry without `where:` and `onRefused` generates exactly as before.

A data store that cannot take back part of a unit of work cannot leave one child out. There, a refused
child under `onRefused: skip` fails the whole source row.

## Prior art / workarounds

A hand-written job that walks the same rows, or a refusal moved out of the target so the run can
finish. The first is the job the format already describes. The second books the rows the target
exists to refuse.

## Specification text

**Anchor:** Declarative glue > schedules. A new subsection after the `generate` example (the one
that fans out `children`), before "The natural key - `generate.unique`". The last normative paragraph
of "The natural key - `generate.unique`" is amended.

New subsection:

#### Which rows become children - `where`, and a refused child - `onRefused`

An `entity` child may take a `where:`: the conditions a schedule's own `where` carries, read on each
row of the collection, and only the rows that satisfy all of them become children. A condition reads a
field or to-one relation of the collection entity, or one relation hop, `<relation>.<field>`. The
collection, and the entity a hop reaches, may each be owned by another model - the hop reads the
related entity in the model that owns it:

```yaml
      children:
        - to: EmployeeTimesheet
          parent: ProjectTimesheet
          forEach: { entity: EmployeeProjectAssignment, model: projects, match: { Project: id } }
          where:
            - { field: Employee.active, op: eq, value: true }   # the employee is the employees model's
          map: { Employee: Employee }
          onRefused: skip
```

A source row's generation is one unit, so by default one child the target refuses fails the header
and every sibling. `onRefused: skip` on a children entry leaves such a child out instead, and the
header and the other children are still created.

> **Normative.**
> A conforming generator MUST create a child only for a collection row that satisfies every condition
> of the entry's `where`, with the operators and values of a schedule's `where`; a hop condition MUST
> be read on the row the relation points at, in the model that owns that row, and a row whose
> relation is empty MUST NOT satisfy it. It MUST reject `where` on a `days` child, a path longer than
> one hop, and a field or relation the entity it is read on does not have - checked against the
> owning model, for an entity another model owns. A seed name on a field of a collection of the same
> model MUST resolve as in a schedule's `where`.
>
> Under `onRefused: skip`, a child the target refuses through its own rules MUST be left out with
> everything written for it beneath it - its writes and the announcements of them taken back - and
> MUST be logged with the source row's key; the header and the other children MUST still be created.
> A refusal beneath a child whose own entry does not declare `onRefused: skip` MUST leave out the
> nearest child that does, or fail the source row when none does. Any other failure MUST fail the
> source row. A header's totals MUST reflect only the children that were created. `skip` is the only
> value; any other MUST be rejected. Where the data store cannot take back part of a unit of work, a
> refused child MUST fail the source row.

Amended (the paragraph "A source row's whole generation - the header, its `children` and theirs - MUST
be one unit of work, ..." in "The natural key - `generate.unique`"), appended sentence:

> A child left out under [`onRefused: skip`](#which-rows-become-children---where-and-a-refused-child---onrefused)
> is taken back within that unit; the rest of the row is not.

## DSL index

The `schedules` row is amended:

| Construct | What it does |
| --- | --- |
| [`schedules`](#schedules) | cron: notify and/or generate records per matching row; `where` values may be moments relative to the firing; a generation's `children` take a row rule (`where`, one relation hop) and may leave out a child the target refuses (`onRefused: skip`) |
