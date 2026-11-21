# A refused create-from is in the reader's language

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8646
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8677

## The problem

A `generates` action refuses a run in several cases: a rule (`items.where`) that qualifies no source
row, an unqualified row under `items.refuse`, a required `prompt` input left empty, a source line the
target refuses, and the values of a report the target cannot hold. Each refusal was written in the
default language only, whatever the reader's language, and named the action and the entities by
their identifiers. The authored `items.refuse` message could not be translated at all: it was not a
translation entry of the model. The other refusals of a translated application - checks, a value of
one field ([0145](0145-field-refusals-name-their-field.md)), a referenced or a master delete
([0141](0141-delete-refusal-lists-every-reference.md), [0146](0146-master-delete-refusal-translated.md))
- are resolved for the reader's language, so a create-from button answered in the default language
on a screen in any other. None of these refusals carried a translation key, so a client could not
tell them from any other validation outcome.

## The proposed shape

No new key. The rule holds for every `generates` action, and the authored message keeps its place:

```yaml
generates:
  - name: invoice-from-month
    from: ProjectMonth
    to: SalesInvoice
    items:
      from: EmployeeTimesheet
      to: SalesInvoiceItem
      where:
        - { field: Status, op: eq, value: APPROVED }
      refuse: "Member timesheet is not approved"
      map: { name: employeeName, quantity: totalHours, price: rate }
    prompt:
      - { field: dueDate, required: true }
```

## Expected behaviour

- Every refusal of a generation is resolved for the reader's language, as a check message is. Each
  sentence is a translation entry of the model, and the action, the source, the target and the
  source's items are named by their translated labels. An entry that a language does not translate
  falls back to the default-language text.
- The authored `items.refuse` message is a translation entry of the model too, so a language
  translates it like a check message; the rows that failed still follow it.
- A required `prompt` input left empty is refused as a value of that one field is (0145): in the
  reader's language, naming the input, so the dialog marks it.
- A source line the target refuses keeps the target's own refusal - already in the reader's
  language, with its key and its field - and names the source row it came from.
- Each refusal carries its translation key and its parameters next to the resolved text, so a
  client can render it again in another language.

## Edge rules

- Nothing about when a generation is refused changes: every refusal is still decided before the
  target header is built, and a refused generation still creates nothing.
- An entity owned by another model (`uses` / `fromUses`) is named by its default-language label
  where the model cannot name its translation entry.

## Prior art / workarounds

None. A translated application showed these refusals in the default language, and an author could
not say the `items.refuse` message in any other. Reference implementation:
https://github.com/eclipse-dirigible/dirigible/pull/8677.

## Specification text

**Anchor:** Declarative glue > generates - create-from > Which source rows become lines -
`items.where` / `items.refuse`, the Normative paragraph. The sentence below is appended to it.

> Every refusal of a generation - the authored `refuse` message included - MUST be resolved for the
> reader's language like a check message, naming the action and the entities by their translated
> labels, and MUST carry its translation key; a required `prompt` input left empty MUST be refused
> as the value of that one field is.
