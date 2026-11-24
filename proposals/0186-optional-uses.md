# A dependency that may be absent - `optional:` on a `uses:` entry

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/7979
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8897

## The problem

A `uses:` entry is all-or-nothing. An expenses model uses a projects model for one feature - a
billable expense line names the project it is re-billed to, a billable line without a project is
refused, a line on a project that is not billable is refused, and a report sums the lines per
project. Every application that ships the expenses model must then ship the projects model, and
everything the projects model uses in turn - customers, sales invoices, products - even an HR
application that never bills anything to a project. Nothing in the format can say "use this model
where the application has it, and do without it where it does not".

## The proposed shape

```yaml
name: expenses
uses:
  - { model: projects, optional: true }
entities:
  - name: ExpenseLine
    checks:
      - { kind: requiredWhen, field: Project, when: "billable == true",
          message: "A billable line needs the project it is billed to" }
      - { kind: forbidWhen, when: ["billable == true", "Project.billable == false"],
          message: "The project is not billable" }
    fields:
      - { name: id,       type: integer, primaryKey: true, generated: true }
      - { name: amount,   type: decimal, required: true }
      - { name: billable, type: boolean, defaultValue: false }
    relations:
      - { name: Project, kind: manyToOne, to: Project, model: projects }
reports:
  - { name: LinesByProject, source: ExpenseLine, dimensions: [Project, billable], measures: ["sum(amount)"] }
```

## Expected behaviour

- Where the optional model is available when the model is generated, nothing changes: the entry
  means what a `uses:` entry always meant.
- Where it is not, the model is generated without it, and the generator reports what it left out:
  - every relation to the absent model - no column, no picker - and every list item that names the
    model by its alias: an `exists` or `notExists` check on one of its entities, a `related`
    register, a `generates` to or from it, a roll-up of its rows, a schedule over them, a report's
    union member;
  - every construct that names such a relation by a path from its own entity: the entity's `order`,
    `list` and `subtitle` entries and its checks (a `requiredWhen` on the relation, a `forbidWhen`
    reading through it), a form's fields, a roll-up `via` the relation, and a report naming it - the
    whole report, since regrouped without that dimension its figures would mean something else.
- In the example, without the projects model a billable line is saved without a project and the
  `LinesByProject` report does not exist; with it, both checks refuse and the report groups by
  project.
- A check that stays keeps the message translation key it has when nothing is left out.

## Edge rules

- `optional` is a boolean; omitted, it is `false`.
- A relation to an optional model is refused at parse when it is `required: true`, part of a
  `unique` key, or the `per` of a number series: without the model the record could not exist, or
  could not be told apart.
- The model must also stand without its optional models. Anything that depends on one and is not
  left out by the rules above - a field's `dependsOn` from such a relation, say - is refused at
  parse, whether or not the optional model is available.
- The decision is made when the model is generated. What was generated with the optional model
  available depends on it like any `uses:`.

## Prior art / workarounds

The dependency shipped with every application that ships the model, or the feature removed from the
model. Reference implementation: https://github.com/eclipse-dirigible/dirigible/pull/8897.

## Specification text

**Anchor:** Entities & fields > Multi-model applications > Reuse, don't redefine — `uses`: a new
subsection *A dependency that may be absent — `optional`* follows the section's Normative block,
before *One shared shell — contributions, not app-hopping*.

#### A dependency that may be absent — `optional`

A model may use another for one feature only - expense lines billed to a project - while some
applications ship it without the other model. Marking the `uses:` entry `optional: true` lets the
model be generated either way: where the other model is available it means what a `uses:` entry
always meant; where it is not, the model is generated without it and without everything that
depends on it, and the generator says what it left out.

```yaml
uses:
  - { model: projects, optional: true }
```

Left out where the optional model is absent: the relations to it, every list item naming it by its
alias (an `exists` / `notExists` check, a `related` register, a `generates`, a roll-up, a schedule, a
report's union member), and every construct naming such a relation by a path from its own entity -
the entity's `order`, `list` and `subtitle` entries and checks, a form's fields, a roll-up `via` it,
and a report, which goes whole.

> **Normative.**
> `optional`, when present, MUST be a boolean; omitted, it is `false`. A relation whose `model:` is
> an optional entry MUST NOT be `required: true`, part of a `unique` key or the `per` of a number
> series. A generator MUST treat an optional model as ABSENT when, at generation, it cannot read
> that model's generated definition, and MUST then generate the declaring model exactly as if it
> declared none of: the `uses:` entry; every list item whose `model`, `uses` or `fromUses` names
> the entry; every relation to the model; and every `order`, `list` or `subtitle` entry, check,
> form field, roll-up `via` and report that names such a relation by a path walked from its own
> entity over relations of the same model. A check that stays MUST keep the message translation key
> it has when nothing is left out. The generator MUST report what it left out, and MUST NOT emit
> anything that refers to the absent model. A model that would not be valid with its optional
> entries left out this way MUST be rejected at parse time, whether or not the models are
> available. Where an optional model is available, a generator MUST treat the entry as a `uses:`
> entry without `optional`.

## DSL index

| Construct | What it does |
| --- | --- |
| [`uses[].optional`](#a-dependency-that-may-be-absent--optional) | a dependency the application may not ship - where its model is absent, the relations to it and the checks, list entries, form fields, roll-ups and reports naming them are left out of the generated model and reported, instead of the generation failing |
