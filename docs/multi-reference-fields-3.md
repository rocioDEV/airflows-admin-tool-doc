### what we changed (frontend)

- **multi-select UX and save behavior**
  - `src/components/entity-view/entity-field/DirectReferenceField/DirectReferenceField.tsx`: multi-select shows
    - always for backend array refs (`attribute.array === true`)
    - only in create mode when backend marks it as multi (`attribute.isMultipleReference === true`)
  - `src/components/entity-view/util/getVariableValue.ts`: serializes
    - array refs → array of ids
    - backend-marked multi refs in create → scalar id (per fan-out row)
    - update accepts accidental array and picks first (robustness)
  - `src/components/entity-view/hooks/useFormValues/useFormValues.ts`: initialize backend-marked multi ref as `[]` in create to avoid empty chip
  - `src/components/entity-view/util/multiReferenceCreate.ts`: detect multi refs using backend `attribute.isMultipleReference` only; fan-out one create per selection combo
  - `src/components/entity-view/hooks/useOnSaveEntity.tsx`: restored the success snackbar with “add another” button for single and fan-out create

- **UI polish for chips**
  - `src/components/ui/autocomplete/components/MultiValue.tsx`: outlined chips (like `TextMultipleField`)
  - `src/components/ui/autocomplete/components/Control.tsx`, `.../ValueContainer.tsx`, `.../AutoComplete.tsx`: increased height, wrap, gaps to prevent chip clipping

- **model parsing and types**
  - `src/components/bootstrap/all-models/parseModels.ts`: no longer propagates `isMultipleReference`; backend provides it on attributes
  - `src/data/model/GetModelType.ts`: optional `isMultipleReference?: boolean` on `DBAttributeDefinition` for typed access

- **tests**
  - `src/components/entity-view/util/__tests__/multiReferenceCreate.spec.ts`: updated to assert backend-only detection of multi refs
  - Full test suite green

- **docs**
  - `airflows-admin-tool-doc/docs/airflows2_0/core/isMultipleReference-on-attribute.md`: detailed plan to expose `isMultipleReference` on attribute objects from backend to remove local injection

### what we changed (backend)

- **flags in the in-memory model**
  - `airflows-uno/com.niledb.platform/src/main/java/data/EntityReference.java` / `.../impl/EntityReferenceImpl.java`: added `isMultipleReference` field and accessors
  - `airflows-uno/com.niledb.platform/src/main/java/data/EntityAttribute.java` / `.../impl/EntityAttributeImpl.java`: added `isMultipleReference` on attributes

- **populate flags while building the model (one bulk query)**
  - `airflows-uno/com.niledb.platform/src/main/java/helpers/DatabaseHelper.java`:
    - Bulk SELECT from `Models.EntityReference` to build `Map<String, Boolean> multiRefByFk` keyed by `schemaname.tablename#refname`
    - Set `reference.isMultipleReference` during the imported-keys loop
    - Mirror the flag to each FK attribute: `attribute.setMultipleReference(true)`

- **getModel**
  - `airflows-uno/com.niledb.platform/src/main/java/graphql/resolvers/GetModelFetcher.java`: no propagation needed; attributes already include `isMultipleReference` in the base model (both super and non‑super)

### frontend cleanup completed

- Removed transitional propagation in `parseModels.ts`
- Stopped falling back to `localModel` in:
  - `DirectReferenceField.tsx`
  - `getVariableValue.ts`
  - `multiReferenceCreate.ts`
  - `useFormValues.ts`
- Removed local augmentation from `ModelEntityAttribute` type
- Ran `pnpm run codegen` in `airflows-admin-tool`
- Tests updated and green; consider small e2e to validate:
  - fan-out create for backend‑marked multi refs
  - single-select display in edit/view
  - array refs remain multi in all modes

### note on performance

- Doing one bulk SELECT for `Models.EntityReference` (all rows) and then O(1) lookups while iterating JDBC imported keys avoids N round-trips (one per FK). This is faster and simpler under large schemas.