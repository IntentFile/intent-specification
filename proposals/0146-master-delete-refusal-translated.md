# A refused master delete is in the reader's language

- **Status:** draft
- **Issue:** https://github.com/eclipse-dirigible/dirigible/issues/8635
- **Implementation:** https://github.com/eclipse-dirigible/dirigible/pull/8641

## The problem

`whenMasterDeleted: refuse` rejects the deletion of a master while any child of the relation exists,
"with a message naming the master and the child entity". The message was written in the default
language only, whatever the reader's language, and the entities were named by their default-language
labels. The other refusal of the same Delete, for a record that other records reference, is
translated ([0141](0141-delete-refusal-lists-every-reference.md)), so a translated application
answered one Delete in two languages depending on why it was refused. The message also carried no
translation key, so a client could not tell it from any other validation outcome.

## The proposed shape

No new key. The rule holds for every composition that already declares `refuse`:

```yaml
- name: CompanyBankAccount
  relations:
    - { name: Company, kind: manyToOne, to: Company, composition: true, required: true, whenMasterDeleted: refuse }
```

## Expected behaviour

- The refusal is resolved for the reader's language, as a check message is. The sentence is a
  translation entry of the model (`This Company still has Company Bank Account records - delete
  those first` in the default language), and the master and the child are named by their translated
  labels. An entry that a language does not translate falls back to the default-language text.
- The refusal carries its translation key and its parameters (the master and the child) next to the
  resolved text, so a client can render it again in another language.

## Edge rules

- Nothing about when the refusal applies changes: it is still decided before the first cascade, and
  it still holds for every writer that can delete the master.
- Where several compositions refuse, the first child entity that still has rows is named, as before.

## Prior art / workarounds

None. A translated application showed this one refusal in the default language. Reference
implementation: https://github.com/eclipse-dirigible/dirigible/pull/8641.

## Specification text

**Anchor:** Relations & multi-model > relations > Relation attributes, the `whenMasterDeleted: refuse`
bullet. Its first sentence is replaced by the one below.

> - **`whenMasterDeleted: refuse`** - deleting the master MUST be rejected while any child of this
>   relation exists, with a message naming the master and the child entity, resolved for the
>   reader's language like a check message, the entities named by their translated labels, and
>   nothing MUST be deleted.
