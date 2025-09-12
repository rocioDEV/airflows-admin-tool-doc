---
title: Expose isMultipleReference on attributes
---

### overview

Goal: expose the `isMultipleReference` flag directly on the attribute objects returned by the model query (`ModelEntity['attributes']`). This removes the need for the frontend to inject the flag into `localModel.entities[...].attributes[...]` in `parseModels.ts`.

Why: today the flag lives at the reference level (DB: `Models.EntityReference.isMultipleReference`) and the frontend carries it into attributes for UI. Moving it into the attribute payload from the backend simplifies the client and avoids duplication.

### current behavior (before)

- Backend
  - DB column exists: `Models.EntityReference.isMultipleReference` (see migration like `airflows-uno/com.niledb.platform/misc/model/sql/deltas/139.sql`).
  - GraphQL exposes the flag on the direct reference node (not on `attributes`).

- Frontend
  - Adds `isMultipleReference?: boolean` to `ModelEntityAttribute` locally:
    - `airflows-admin-tool/src/components/dashboard/refresh/util/types.ts`
  - Injects the flag into `localModel.entities[...].attributes[...]` inside `parseModels.ts`:
    - `airflows-admin-tool/src/components/bootstrap/all-models/parseModels.ts`
  - Consumers read the flag from `localModel` (e.g., `DirectReferenceField.tsx`).

### target behavior (after)

- Backend returns `isMultipleReference` directly on each attribute object when that attribute represents a direct reference.
- Frontend stops injecting this flag into `localModel` and reads it straight from the attribute payload (`DBAttributeDefinition`).

### backend changes

1) GraphQL schema (SDL)

- Update the SDL so that the attribute type returned in `entities { attributes { ... } }` includes `isMultipleReference: Boolean`.
  - Edit: `airflows-admin-tool/sdl.gql` (frontend copy used for codegen) and the backend SDL in the server repo (source of truth).
  - Example (illustrative):

```graphql
type Models_EntityAttribute {
  name: String!
  type: String!
  # existing fields...
  referencedKey: Models_EntityKey
  # new field
  isMultipleReference: Boolean
}
```

2) Resolver / model assembler

- Wherever the server builds `entities.attributes`:
  - Identify attributes that represent a direct reference (those that link to a `Models.EntityReference`).
  - Look up the corresponding `EntityReference.isMultipleReference` for that attribute.
  - Set `attribute.isMultipleReference = entityReference.isMultipleReference` in the returned object.

Notes:
- Make sure this value is present for the attribute named after the reference attribute (the same name used in `ModelEntity['attributes']`).
- Do not set it for non-reference attributes.

3) Backward compatibility

- Keep the flag available on the direct reference node if it already exists to avoid breaking clients; this task focuses on adding it to `attributes`.

### frontend changes

1) regenerate types

- After updating `sdl.gql`, run codegen to add the new field to generated types:
  - `airflows-admin-tool/package.json` script: `pnpm run codegen`
  - Verify `DBAttributeDefinition` (or the corresponding generated type the app uses for attributes) now includes `isMultipleReference?: boolean`.

2) stop injecting flag in localModel

- In `parseModels.ts`, remove the logic that copies reference-level `isMultipleReference` into `localModel.entities[...].attributes[...]`.
  - File: `airflows-admin-tool/src/components/bootstrap/all-models/parseModels.ts`
  - Remove the block that sets `localModelEntity.attributes[referenceAttributeName].isMultipleReference = ...`.

3) update consumers to read from attribute directly

- `DirectReferenceField.tsx`:
  - Today it reads from `localModel.entities[entity].attributes[...]` to decide multi UI. Change to read from the `attribute` prop (DB attribute) when determining `isMultipleReference`.
  - Keep behavior distinctions:
    - Backend array refs (`attribute.array === true`) → always multi.
    - Local-only multi via `attribute.isMultipleReference === true` → multi only in create (no `entityId`).
  - File: `airflows-admin-tool/src/components/entity-view/entity-field/DirectReferenceField/DirectReferenceField.tsx`

- `getVariableValue.ts`:
  - Where it currently checks local-only multi to decide scalar vs array on save, switch to use `attribute.isMultipleReference` instead of reading from `localModel`.
  - File: `airflows-admin-tool/src/components/entity-view/util/getVariableValue.ts`

- `multiReferenceCreate.ts`:
  - `getLocalMultipleReferenceAttributes` can be simplified/renamed to `getMultipleReferenceAttributes` and use the attribute’s own `isMultipleReference` flag (ignoring `localModel`).
  - File: `airflows-admin-tool/src/components/entity-view/util/multiReferenceCreate.ts`

4) types cleanup

- Remove the augmentation from `ModelEntityAttribute` once all reads are from DB attribute data:
  - File: `airflows-admin-tool/src/components/dashboard/refresh/util/types.ts`
  - Delete `isMultipleReference?: boolean`.

5) transitional compatibility (optional but recommended)

- For one release, keep a fallback read from `localModel` if `attribute.isMultipleReference` is `undefined`, to remain compatible with older servers:
  - `DirectReferenceField.tsx` and `getVariableValue.ts` can use `attribute.isMultipleReference ?? localModelFallback`.

### tests

- Unit tests
  - Add tests for `DirectReferenceField` logic deciding when `isMulti` is true based on:
    - `attribute.array === true` (always multi)
    - `attribute.isMultipleReference === true` with create vs edit modes
  - Add tests covering `getVariableValue` for reference attributes with:
    - array refs → array of ids
    - local-only multi in create fan-out → scalar id per create

- Integration/e2e
  - Create entity with a reference marked multi → verify fan-out creates one row per selection.
  - Edit/view created entity → UI shows single select for local-only multi.

### rollout plan

1) Backend deploy
  - Add SDL field and server-side mapping.
  - Deploy backend first.

2) Frontend update
  - Run `pnpm run codegen`.
  - Implement client changes reading from `attribute.isMultipleReference`.
  - Remove local injection in `parseModels.ts`.
  - Keep one-release fallback (optional).

3) Clean up
  - After one release cycle with both sources, remove any fallbacks and delete the `ModelEntityAttribute` augmentation.

### potential pitfalls

- Mismatch between attribute names and reference mapping: ensure the attribute receiving the flag is the same attribute rendered in forms (check `referenceAttributeName`).
- Cached model data: invalidate/refresh model queries after deployment so clients receive the new field.
- Codegen drift: always regenerate types after SDL changes to avoid `any` leaks.

### file references (summary)

- Backend DB: `airflows-uno/com.niledb.platform/misc/model/sql/deltas/139.sql`
- Front SDL for codegen: `airflows-admin-tool/sdl.gql`
- Front types augmentation: `airflows-admin-tool/src/components/dashboard/refresh/util/types.ts`
- Front model parsing: `airflows-admin-tool/src/components/bootstrap/all-models/parseModels.ts`
- Front UI field: `airflows-admin-tool/src/components/entity-view/entity-field/DirectReferenceField/DirectReferenceField.tsx`
- Front value serializer: `airflows-admin-tool/src/components/entity-view/util/getVariableValue.ts`
- Front multi create util: `airflows-admin-tool/src/components/entity-view/util/multiReferenceCreate.ts`


