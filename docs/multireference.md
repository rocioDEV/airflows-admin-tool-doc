
## multi-select direct references in entity view (implementation guide)

### summary

- **goal**: allow selecting multiple values in direct reference attributes when creating/editing an entity in the admin tool `EntityView`.
- **storage**: if the FK attribute is an array type (e.g., `INTEGER[]`, `TEXT[]`) and the GraphQL input accepts lists (`[Int]`/`[String]`), persist the list of referenced PKs directly. If the relationship is modeled via a join entity, manage link rows instead (see backend verification).
- **scope**: UI changes (multi-select), form value init, value mapping for mutations, optional default-values recalculation, tests, and docs.

### backend verification checklist (array fk vs join entity)

Run these checks for each target attribute you want to make multi-select:

- **database schema** (airflows-uno)
  - Confirm the owning entity table column is an array type (e.g., `INTEGER[]`, `TEXT[]`). If it’s scalar (e.g., `INTEGER`), you cannot directly store multiple references in that column.
  - Where to look: migrations/DDL. Example of array usage in the platform DDL (illustrative, not the FK itself):
    - `/home/rderandom/dev/airflows-uno/com.niledb.platform/misc/model/create.sql`
      - shows array-typed columns and type suffixing with `[]` in some places (e.g., `"acceptedFileTypes" text[]`).

- **generated GetModel / frontend model** (airflows-admin-tool)
  - The attribute must have both:
    - `referenceAttributeName` defined (so it’s a direct reference)
    - `array === true` (so it’s multi)
  - Types reference: `src/data/model/GetModelType.ts` in `/home/rderandom/dev/airflows-admin-tool/` (interface `DBAttributeDefinition`).

- **GraphQL SDL input** (airflows-admin-tool)
  - In `sdl.gql`, confirm the entity’s `CreateInputType` and `UpdateInputType` expose the FK field as a list (`[Int]` or `[String]`). If it’s `Int`/`String` only, the API does not accept arrays for that attribute yet.
  - File: `/home/rderandom/dev/airflows-admin-tool/sdl.gql`.

- **absence of join entity**
  - If there is a normalized join entity modeling A↔B, neither side will have an array FK. In that case, do not attempt to write `[PK]` into the parent; instead, manage link rows via the join entity (see alternative plan).

### implementation plan (array fk path)

1) **ui changes: multi-select autocomplete**

- File: `/home/rderandom/dev/airflows-admin-tool/src/components/ui/autocomplete/AutoComplete.tsx`
  - Add prop `isMulti?: boolean`.
  - Relax `value` and `onChange` types to support arrays: `value?: T | T[]`, `onChange: (value: T | T[]) => void`.
  - Pass `isMulti` to `AsyncSelect` (`isMulti={isMulti}`).
  - Hide the single-record “open” icon when `isMulti` is true; keep the “manage” list icon.

- File: `/home/rderandom/dev/airflows-admin-tool/src/components/entity-view/entity-field/DirectReferenceField/DirectReferenceField.tsx`
  - Remove the early return that disables arrays:
    - Current: `if (attribute.array) { return null }`.
  - When `attribute.array === true`, render `<AutoComplete isMulti ... />`.
  - `onChange` must forward an array of option objects to react-hook-form’s `field.onChange`.
  - Remove `key={field.value}` to avoid remounts (breaks for arrays).
  - Keep support for `additionalFilter` and `additionalAttributes` as-is.

2) **form value initialization**

- File: `/home/rderandom/dev/airflows-admin-tool/src/components/entity-view/hooks/useFormValues/useFormValues.ts`
  - Add branch for array direct references:
    - If `attribute.array && attribute.referenceAttributeName`, expect `entityStateData[attribute.referenceAttributeName]` to be a list of referenced objects.
    - Map each to `{ value: <subKey PK>, label: getLabel(...) }` (similar to existing `buildReferenceAttribute`, but for arrays).
  - Keep single-select initialization unchanged.

3) **persistence mapping (create/update)**

- File: `/home/rderandom/dev/airflows-admin-tool/src/components/entity-view/util/getVariableValue.ts`
  - For references where `attribute.referenceAttributeName && attribute.array === true`:
    - Convert `formValue` (array of `{ value, label }`) into an array of PKs.
    - Return the correctly typed array based on `attribute.type` (numeric vs string primary key).
    - Return `null`/`undefined` on empty selection per current backend semantics.

- Files: `/home/rderandom/dev/airflows-admin-tool/src/data/mutations/createEntity.ts`, `/home/rderandom/dev/airflows-admin-tool/src/data/mutations/updateEntity.ts`
  - No structural changes needed: they already use `getVariableType` which respects `attribute.array` and will produce `[Int]`/`[String]` variable types.

- File: `/home/rderandom/dev/airflows-admin-tool/src/data/util/getVariableType.ts`
  - Already handles arrays (`attribute.array` adds surrounding `[]`). No changes required.

4) **default values recomputation (optional)**

- File: `/home/rderandom/dev/airflows-admin-tool/src/components/entity-view/entity-field/DirectReferenceField/useRecalculateDefaultValues.ts`
  - Enhance `hasChangedValue` to support arrays: compare selection sets by option `.value` (PKs) rather than object identity, to avoid unnecessary recomputations.
  - If backend default-values function does not accept arrays, skip recalculation for multi-ref parameters.

5) **display labels and fetching**

- File: `/home/rderandom/dev/airflows-admin-tool/src/components/entity-view/hooks/useLoadEntityState/mapAttributesToQuery.ts`
  - No change required. The GraphQL shape `referenceAttributeName { pk, labels... }` also works when the field is a list (server returns a list of objects).

- File: `/home/rderandom/dev/airflows-admin-tool/src/components/ui/autocomplete/hooks/useRefreshData.tsx`
  - No change required to option fetching. Optionally guard single-value hydration logic when `isMulti` is true.

6) **tests**

- e2e scenarios (add or extend under `/home/rderandom/dev/airflows-admin-tool/e2e-tests/`):
  - Create: select multiple referenced records; intercept GraphQL mutation; assert variable is an array of PKs; reload and verify selections.
  - Update: add/remove selections and verify persisted result.
  - Filters: validate `additionalFilter` still applies to option loading.
  - Edge cases: empty selection, many selections, slow network.

7) **docs**

- Update docs to reflect multi-select behavior for direct references:
  - File: `/home/rderandom/dev/airflows-admin-tool-doc/docs/airflows2_0/core/data-model.md`
  - File: `/home/rderandom/dev/airflows-admin-tool-doc/docs/airflows2_0/core/1localModel.md`

### alternative plan (join entity path)

If the relationship is normalized via a join entity (no array FK on the parent):

- **ui**: keep the multi-select UX in `EntityView`.
- **read**: load the join entity list filtered by the current entity id to hydrate selected options.
- **save**: compute delta between previous and new selections; issue mutations to create/delete link rows in the join entity, instead of sending an array into the parent.
- **pros/cons**: preserves referential integrity and supports extra relationship fields; requires multiple mutations and one additional query layer.

### risks and mitigations

- **schema mismatch**: UI assumes array FK but DB/SDL is scalar → follow the verification checklist; if mismatch, switch to join-entity path.
- **icon behavior**: single-record “open” icon is ambiguous with multiple selections → hide when `isMulti` is true, keep the “manage” list icon.
- **performance**: large selected sets and large option lists → consider virtualization in autocomplete if needed.
- **default-values**: ensure multi-select parameters are handled by backend or are skipped to avoid noisy recomputation.

### acceptance criteria

- When `DBAttributeDefinition.referenceAttributeName` is defined and `DBAttributeDefinition.array === true`, `EntityView` renders a multi-select autocomplete.
- Create/Update mutations receive arrays of PKs for these attributes (array FK path), or link rows are created/removed (join entity path).
- Reopening an entity shows the same selected set.
- No regressions for single-reference fields.
- `additionalFilter` and `additionalAttributes` continue working.

### file references to update (summary)

- `src/components/ui/autocomplete/AutoComplete.tsx`
- `src/components/entity-view/entity-field/DirectReferenceField/DirectReferenceField.tsx`
- `src/components/entity-view/hooks/useFormValues/useFormValues.ts`
- `src/components/entity-view/util/getVariableValue.ts`
- `src/components/entity-view/entity-field/DirectReferenceField/useRecalculateDefaultValues.ts` (optional)
- `src/components/entity-view/hooks/useLoadEntityState/mapAttributesToQuery.ts` (read-only confirm)
- `docs/airflows2_0/core/data-model.md`, `docs/airflows2_0/core/1localModel.md`

### rollout plan

1) Verify backend storage model using the checklist.
2) Implement UI/value mapping edits (array FK path).
3) Add tests for create/update and readback.
4) If verification fails (join entity), switch to the alternative plan for persisting via link rows.
5) Update docs.