---
sidebar_position: 2
---

# Sistema de privilegios

## Descripción general

El sistema de privilegios es un mecanismo de seguridad integral que controla el acceso, visibilidad y capacidad de edición de los campos basándose en los permisos del usuario. Implementa seguridad a nivel de fila verificando privilegios específicos para cada operación (SELECT, INSERT, UPDATE, DELETE) tanto a nivel de entidad como de atributo. Este sistema asegura que los usuarios solo puedan acceder y modificar datos para los que tienen permiso.

## Conceptos principales

### Tipos de privilegios

El sistema soporta cinco tipos principales de privilegios:

```typescript
privileges?: {
  SELECT?: string
  INSERT?: string
  UPDATE?: string
  DELETE?: string
  MENU?: string
}
```

- **SELECT**: Permiso para ver/leer el campo
- **INSERT**: Permiso para crear nuevos registros con este campo
- **UPDATE**: Permiso para modificar registros existentes con este campo
- **DELETE**: Permiso para eliminar registros
- **MENU**: Permiso para acceder a la entidad en menús

### Bypass de super usuario

Los usuarios con `model.super = true` evitan todas las verificaciones de privilegios y tienen acceso completo a todos los campos y operaciones.

### Privilegios de entidad vs atributo

Los privilegios pueden definirse en ambos niveles:
- **Privilegios de entidad**: Se aplican a todos los atributos de una entidad
- **Privilegios de atributo**: Sobrescriben los privilegios de entidad para campos específicos

## Definiciones de tipos

```typescript
// En el tipo Attribute
privileges?: {
  SELECT?: string
  INSERT?: string
  UPDATE?: string
  DELETE?: string
  MENU?: string
}

// En el tipo DBEntityDefinition
privileges: {
  DELETE?: string
  UPDATE?: string
  MENU?: string
  INSERT?: string
  SELECT?: string
}
```

## Cómo funciona el sistema de privilegios

### 1. Filtrado de visibilidad de campos

Durante la carga de entidades, los atributos se filtran basándose en privilegios SELECT:

```typescript
const attributes = Object.values(entityModel.attributes)
  .filter((attribute) => model.super || attribute.privileges?.['SELECT'] !== undefined)
  .map((attribute) => {
    // ... mapeo de atributos
  })
```

**Lógica**:
- Si `model.super` es true, se incluyen todos los atributos
- De lo contrario, solo se incluyen atributos con privilegios SELECT definidos
- Los atributos sin privilegios SELECT se ocultan completamente del usuario

### 2. Estado deshabilitado de campos

Los campos se deshabilitan basándose en privilegios INSERT/UPDATE y estado de la entidad:

```typescript
const disabled = 
  (!isEntityEmpty && attribute.editableOnInsertOnly) ||
  attribute.computed ||
  mode === 'view' ||
  (!model?.super &&
    ((!isEntityEmpty && attribute.privileges?.['UPDATE'] === undefined) ||
     (isEntityEmpty && attribute.privileges?.['INSERT'] === undefined)))
```

**Lógica**:
- Para entidades existentes (`!isEntityEmpty`): Requiere privilegio UPDATE
- Para entidades nuevas (`isEntityEmpty`): Requiere privilegio INSERT
- Los super usuarios evitan las verificaciones de privilegios
- Condiciones adicionales: campos computados, modo vista, editableOnInsertOnly

### 3. Validación de atributos

La función `isInvalidAttribute` verifica si un atributo debe incluirse en las operaciones:

```typescript
export function isInvalidAttribute(isSuper: boolean, attribute: DBAttributeDefinition, entityId?: string) {
  return (
    !isSuper &&
    ((!entityId && attribute.privileges?.['UPDATE'] === undefined) ||
      (entityId && attribute.privileges?.['INSERT'] === undefined))
  )
}
```

**Lógica**:
- Los super usuarios siempre pueden acceder a los atributos
- Para entidades nuevas: Requiere privilegio INSERT
- Para entidades existentes: Requiere privilegio UPDATE

### 4. Filtrado de creación de entidades

Al crear entidades, los atributos se filtran basándose en privilegios INSERT:

```typescript
function filterAttributesForCreate(
  model: DBParsedModel | undefined,
): (value: Attribute, index: number, array: Attribute[]) => unknown {
  return (attribute) =>
    attribute.incomingReferenceAttributeName === undefined &&
    !(!model?.super && attribute.privileges?.['INSERT'] === undefined)
}
```

**Lógica**:
- Excluye atributos de referencia entrante
- Excluye atributos sin privilegios INSERT (a menos que sea super usuario)
- Solo incluye atributos que el usuario puede insertar realmente

## Funciones de verificación de privilegios

### 1. isAttributeDisabled

Función integral que determina si un campo debe estar deshabilitado:

```typescript
export function isAttributeDisabled({
  mode,
  entity,
  attribute,
  entityId,
  model,
  localAttribute,
}: {
  mode: string
  entity: string
  attribute: DBAttributeDefinition
  entityId?: string
  model: DBParsedModel | undefined
  localAttribute?: ModelEntityAtrribute | undefined
}) {
  return !!(
    (localAttribute && localAttribute.computed) ||
    mode === 'view' ||
    (entity === 'Models.EntityReference' && attribute.name === 'container') ||
    (entity === 'Models.EntityReference' && attribute.name === 'referencedKey') ||
    (!model?.super &&
      ((entityId && attribute.privileges?.['UPDATE'] === undefined) ||
        (entityId && localAttribute?.editableOnInsertOnly) ||
        (!entityId && attribute.privileges?.['INSERT'] === undefined)))
  )
}
```

**Casos especiales**:
- Los campos computados siempre están deshabilitados
- El modo vista deshabilita todos los campos
- Los campos container y referencedKey de EntityReference siempre están deshabilitados
- Las verificaciones de privilegios solo se aplican a usuarios no super

### 2. hasInvalidPrivileges

Usado específicamente para campos externos:

```typescript
function hasInvalidPrivileges(
  entityId: string | undefined,
  attribute: DBAttributeDefinition,
  model?: DBParsedModel,
): boolean {
  return (
    !model?.super &&
    ((entityId && attribute.privileges?.['UPDATE'] === undefined) ||
      (!entityId && attribute.privileges?.['INSERT'] === undefined))
  )
}
```

**Uso**: Controla la capacidad de edición y funcionalidad de búsqueda de campos externos

### 3. canEdit

Determina si la acción de editar debe estar disponible:

```typescript
function canEdit(
  mode: string,
  model: DBParsedModel,
  entity: string,
  getEntityFormValues: UseFormGetValues<FieldValues>,
) {
  return (
    mode === 'view' &&
    (model.super || model.entities[entity].privileges['UPDATE'] !== undefined) &&
    (entity !== 'Models.EntityAttribute' || getEntityFormValues('name') !== 'id')
  )
}
```

**Lógica**:
- Solo disponible en modo vista
- Requiere privilegio UPDATE a nivel de entidad o estatus de super usuario
- Caso especial: El campo 'id' de EntityAttribute no puede ser editado

## Integración con tipos de campos

### 1. Campos de texto

```typescript
// En TextField.tsx
disabled={
  (entityId && attribute.editableOnInsertOnly) ||
  attribute.computed ||
  !attribute.externalSelectorEditableValue ||
  mode === 'view' ||
  hasInvalidPrivileges(entityId, attribute, model)
}
```

### 2. Campos booleanos

```typescript
// En BooleanField.tsx
disabled={
  (!isEntityEmpty && attribute.editableOnInsertOnly) ||
  attribute.computed ||
  mode === 'view' ||
  (!model.super &&
    ((!isEntityEmpty && attribute.privileges?.['UPDATE'] === undefined) ||
      (isEntityEmpty && attribute.privileges?.['INSERT'] === undefined)))
}
```

### 3. Campos externos

```typescript
// En ExternalField.tsx
disabled={
  (entityId && attribute.editableOnInsertOnly) ||
  attribute.computed ||
  !attribute.externalSelectorEditableValue ||
  mode === 'view' ||
  hasInvalidPrivileges(entityId, attribute, model)
}
```

### 4. Campos de punto

```typescript
// En PointField.tsx
disabled={
  (!isEntityEmpty && attribute.editableOnInsertOnly) ||
  attribute.computed ||
  mode === 'view' ||
  (!model.super &&
    ((!isEntityEmpty && attribute.privileges?.['UPDATE'] === undefined) ||
      (isEntityEmpty && attribute.privileges?.['INSERT'] === undefined)))
}
```

## Esquema de base de datos

### Esquema GraphQL

```graphql
enum TablePrivilegeType {
  SELECT
  INSERT
  UPDATE
  DELETE
}
```

### Privilegios de atributos de entidad

Los privilegios se almacenan como parte de la definición del atributo de entidad y pueden consultarse a través de mutaciones y consultas GraphQL.

## Patrones de uso

### 1. Definición básica de privilegios

```typescript
const attribute: DBAttributeDefinition = {
  name: 'sensitiveField',
  type: 'TEXT',
  privileges: {
    SELECT: 'user_role',
    UPDATE: 'admin_role',
    INSERT: 'admin_role'
  },
  // ... otras propiedades
}
```

### 2. Campos de solo lectura

```typescript
const attribute: DBAttributeDefinition = {
  name: 'readOnlyField',
  type: 'TEXT',
  privileges: {
    SELECT: 'user_role'
    // Sin privilegios INSERT/UPDATE = solo lectura
  },
  // ... otras propiedades
}
```

### 3. Campos solo para administradores

```typescript
const attribute: DBAttributeDefinition = {
  name: 'adminField',
  type: 'TEXT',
  privileges: {
    SELECT: 'admin_role',
    INSERT: 'admin_role',
    UPDATE: 'admin_role'
  },
  // ... otras propiedades
}
```

### 4. Privilegios a nivel de entidad

```typescript
const entity: DBEntityDefinition = {
  name: 'SensitiveEntity',
  privileges: {
    SELECT: 'user_role',
    INSERT: 'admin_role',
    UPDATE: 'admin_role',
    DELETE: 'admin_role',
    MENU: 'user_role'
  },
  // ... otras propiedades
}
```

## Consideraciones de seguridad

### 1. Validación del lado del servidor

- Las verificaciones de privilegios deben implementarse en el lado del servidor
- Las verificaciones del lado del cliente son solo para UX y pueden ser evadidas
- Todas las operaciones GraphQL deben validar privilegios

### 2. Denegación por defecto

- Los campos sin privilegios explícitos tienen acceso denegado
- El sistema sigue un modelo de seguridad "denegar por defecto"
- Los super usuarios son la única excepción a las verificaciones de privilegios

### 3. Herencia de privilegios

- Los privilegios de entidad proporcionan permisos base
- Los privilegios de atributo pueden sobrescribir los privilegios de entidad
- Los privilegios de atributo más restrictivos tienen precedencia

### 4. Consideraciones de auditoría

- Las verificaciones de privilegios deben registrarse para auditoría de seguridad
- Los intentos fallidos de privilegios deben ser monitoreados
- Los cambios de privilegios deben ser rastreados

## Manejo de errores

### 1. Privilegios faltantes

- Los campos sin privilegios requeridos se ocultan o deshabilitan
- No se muestran mensajes de error a los usuarios por violaciones de privilegios
- El sistema degrada graciosamente la funcionalidad

### 2. Definiciones de privilegios inválidas

- Los objetos de privilegios malformados se tratan como sin privilegios
- El sistema por defecto deniega el acceso
- El registro debe capturar errores de definición de privilegios

### 3. Manejo de super usuario

- Los super usuarios evitan todas las verificaciones de privilegios
- Esto debe usarse con moderación y auditarse cuidadosamente
- El estatus de super usuario debe indicarse claramente en los registros

## Consideraciones de rendimiento

### 1. Sobrecarga de verificación de privilegios

- Las verificaciones de privilegios ocurren en cada renderizado de campo
- Considera cachear resultados de privilegios para formularios complejos
- Agrupa verificaciones de privilegios donde sea posible

### 2. Filtrado de atributos

- El filtrado ocurre durante la carga de entidades
- Un gran número de atributos puede impactar el rendimiento
- Considera carga diferida para secciones dependientes de privilegios

### 3. Gestión de estado

- El estado de privilegios debe cachearse en el contexto
- Evita recalcular privilegios en cada renderizado
- Usa memorización para verificaciones de privilegios costosas

## Mejores prácticas

### 1. Diseño de privilegios

- Define privilegios en el nivel apropiado (entidad vs atributo)
- Usa asignaciones de privilegios basadas en roles
- Mantén definiciones de privilegios simples y consistentes

### 2. Seguridad

- Siempre valida privilegios en el lado del servidor
- Usa el principio de menor privilegio
- Audita regularmente las asignaciones de privilegios

### 3. Experiencia de usuario

- Proporciona retroalimentación clara cuando las acciones no están permitidas
- Considera mostrar campos deshabilitados con tooltips explicativos
- Mantén comportamiento consistente a través de diferentes tipos de campos

### 4. Mantenimiento

- Documenta claramente los requisitos de privilegios
- Usa convenciones de nomenclatura consistentes para roles de privilegios
- Revisa y actualiza regularmente las asignaciones de privilegios 