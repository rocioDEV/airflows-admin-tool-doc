---
sidebar_position: 3
---

# Campos tab y group

## Descripción general

Los campos `tab` y `group` se utilizan en el componente EntityView para organizar y estructurar la visualización de los atributos de una entidad. Proporcionan un sistema de diseño jerárquico que permite agrupar atributos dentro de pestañas, creando una interfaz más organizada y fácil de usar.

## Definiciones de tipo

```typescript
// En el tipo Attribute
tab: string

// En el tipo TabAndGroup
export type TabAndGroup = {
  tab: number | string | undefined | null
  group: number | string | undefined | null
}
```

## Propósito

Estos campos sirven para:

- **Organizar atributos**: Agrupar campos relacionados para una mejor experiencia de usuario
- **Crear interfaces con pestañas**: Permitir que entidades complejas se muestren en múltiples pestañas
- **Proporcionar jerarquía visual**: Usar grupos dentro de pestañas para organizar mejor el contenido
- **Controlar visibilidad**: Determinar qué atributos se muestran según la selección actual de pestaña/grupo

## Estructura

### Pestañas

Las pestañas representan la organización de nivel superior de los atributos de la entidad. Cada pestaña puede contener:
- Atributos sin asignación de grupo
- Múltiples grupos que contienen atributos relacionados

### Grupos

Los grupos representan un nivel secundario de organización dentro de las pestañas. Proporcionan:
- Separación visual de campos relacionados
- Contenedores estilizados con bordes y colores de fondo
- Etiquetas localizadas para una mejor experiencia de usuario

## Patrones de uso

### 1. Atributos sin pestaña ni grupo

```typescript
const attribute: DBAttributeDefinition = {
  name: 'campoBasico',
  type: 'string',
  // Sin pestaña ni grupo especificados - aparece en el nivel superior
}
```

### 2. Atributos solo con pestaña

```typescript
const attribute: DBAttributeDefinition = {
  name: 'campoConPestana',
  type: 'string',
  tab: '1', // Aparece en la pestaña con ID 1
}
```

### 3. Atributos solo con grupo

```typescript
const attribute: DBAttributeDefinition = {
  name: 'campoConGrupo',
  type: 'string',
  group: '2', // Aparece en el grupo con ID 2 (sin pestaña)
}
```

### 4. Atributos con pestaña y grupo

```typescript
const attribute: DBAttributeDefinition = {
  name: 'campoComplejo',
  type: 'string',
  tab: '1',    // Aparece en la pestaña con ID 1
  group: '2',  // Dentro del grupo con ID 2
}
```

## Lógica de renderizado

### Estructura del componente EntityView

El componente EntityView renderiza los atributos en el siguiente orden:

1. **Sin pestaña, sin grupo**: Los atributos aparecen en el nivel superior
2. **Sin pestaña, con grupo**: Los atributos aparecen en contenedores de grupo estilizados
3. **Con pestaña, sin grupo**: Los atributos aparecen directamente en el contenido de la pestaña
4. **Con pestaña, con grupo**: Los atributos aparecen en grupos dentro de las pestañas

### Condiciones de visibilidad

Los atributos solo se renderizan cuando:

```typescript
// Para atributos solo con pestaña
attribute.tab === selectedTab

// Para atributos solo con grupo
attribute.group === currentGroup

// Para atributos con pestaña + grupo
attribute.tab === selectedTab && attribute.group === currentGroup
```

## Lógica de coincidencia

### Función matchesGroupAndTab

Ubicada en `src/components/entity-view/util/matchesGroupAndTab.ts`, esta función determina si un atributo debe mostrarse:

```typescript
export function matchesGroupAndTab(
  attribute?: Partial<TabAndGroup> | null,
  currentTab: number | undefined | null = undefined,
  currentGroup: number | undefined | null = undefined,
) {
  if (!attribute) {
    return false
  }
  const attributeTab = attribute.tab ?? undefined
  const attributeGroup = attribute.group ?? undefined

  return attributeTab === currentTab && attributeGroup === currentGroup
}
```

### Conversión de tipos

La función maneja tanto tipos string como number para los valores de pestaña y grupo, permitiendo formatos de datos flexibles.

## Comportamiento de la UI

### Renderizado de pestañas

- Las pestañas solo se renderizan si la entidad tiene atributos con asignaciones de pestaña
- Las etiquetas de las pestañas se localizan usando la clave de traducción: `e.{entity}.tabs.{tabId}`
- La selección de pestañas se gestiona a través del estado `selectedTab`

### Renderizado de grupos

- Los grupos se renderizan como contenedores estilizados con bordes punteados
- Las etiquetas de los grupos se localizan usando la clave de traducción: `e.{entity}.groups.{groupId}`
- Los grupos respetan la propiedad `order` para el ordenamiento

### Estilos

Los grupos se estilizan con:
```css
{
  backgroundColor: '#f8f8f8',
  borderWidth: 1,
  borderStyle: 'dashed',
  borderColor: '#dddddd',
  padding: '12px',
  margin: 12
}
```

## Esquema de base de datos

### Estructura de pestaña

```typescript
export type ModelEntityTab = {
  id?: number | null
  name?: string | null
  order?: number | null
  labels?: Array<{
    locale?: string | null
    label?: string | null
  } | null> | null
}
```

### Estructura de grupo

```typescript
export type ModelEntityGroup = {
  id?: number | null
  name?: string | null
  order?: number | null
  labels?: Array<{
    locale?: string | null
    label?: string | null
  } | null> | null
}
```

## Funciones relacionadas

### isTabGroupWithAttributes

Ubicada en `src/components/entity-view/util/isTabGroupWithAttributes.ts`, esta función determina si un grupo debe mostrarse dentro de una pestaña específica:

```typescript
export function isTabGroupWithAttributes(
  group: ModelEntityGroup,
  tab: ModelEntityTab,
  entityLocalModel: EntityConfig,
  selectedVariants: string[],
) {
  return (
    entityLocalModel.attributes &&
    Object.values(entityLocalModel.attributes).filter((attribute) => {
      return (
        attribute.group === group?.id &&
        attribute.tab === tab?.id &&
        // Lógica de filtrado de variantes
      )
    }).length > 0
  )
}
```

### localEntityHasAttributesWithTags

Determina si una entidad tiene algún atributo con asignaciones de pestaña:

```typescript
function localEntityHasAttributesWithTags(entityLocalModel: EntityConfig) {
  return (
    entityLocalModel.attributes &&
    Object.values(entityLocalModel.attributes).filter((attribute) => attribute?.tab).length > 0
  )
}
```

## Interacciones importantes con otros componentes

### 1. Integración con el sistema de variantes

El sistema de pestañas y grupos se integra con el sistema de variantes para controlar la visibilidad:

```typescript
// En la función getVisibleTabs
const tabsAttributes = entityLocalModel.attributes &&
  Object.values(entityLocalModel.attributes).filter(
    (attribute) =>
      tab &&
      attribute?.tab === tab.id &&
      (!attribute?.variants ||
        attribute?.variants?.length === 0 ||
        (selectedVariants &&
          selectedVariants.some((variant) => attribute?.variants?.includes('' + variant)))),
  )
```

**Impacto**: Las pestañas solo son visibles si contienen atributos que coinciden con la selección actual de variantes.

### 2. Integración con campos externos

Los campos externos respetan las asignaciones de pestaña y grupo:

```typescript
// En EntityViewFieldGroup.tsx
const isExternalSearch =
  attribute.externalSelectorUrl && attributeTab === tab && attributeGroup === group
```

**Impacto**: Los campos de selector externo solo se renderizan cuando su pestaña y grupo coinciden con el contexto actual.

### 3. Integración con el sistema de diseño de filas

Las propiedades `firstInRow` y `lastInRow` funcionan en conjunto con pestaña y grupo:

```typescript
// En EntityViewFieldGroup.tsx
if (item && item.lastInRow && matchesGroupAndTab(item, tab, group)) {
  // Insertar salto de fila después de este atributo
}

if (item && item.firstInRow && matchesGroupAndTab(item, tab, group)) {
  // Insertar salto de fila antes de este atributo
}
```

**Impacto**: Los saltos de fila solo se aplican cuando la pestaña y grupo del atributo coinciden con el contexto actual.

### 4. Integración con referencias inversas

Las referencias inversas respetan la visibilidad de pestaña y grupo:

```typescript
// En isInverseReference.ts
matchesGroupAndTab(
  entityLocalModel?.attributes?.[attribute.incomingReferenceAttributeName],
  tab,
  group,
)
```

**Impacto**: Las listas de referencias inversas solo se muestran cuando la pestaña y grupo de su atributo asociado coinciden con el contexto actual.

### 5. Integración con el análisis de modelos

Durante el análisis de modelos, las asignaciones de pestaña y grupo se procesan tanto para referencias directas como inversas:

```typescript
// En parseModels.ts - Referencias directas
localModelEntity.attributes[attributeName] = {
  // ... otras propiedades
  tab: reference.tab,
  group: reference.group,
  // ... otras propiedades
}

// En parseModels.ts - Referencias inversas
localModelEntity.attributes[attributeName] = {
  // ... otras propiedades
  tab: reference.tab,
  group: reference.group,
  // ... otras propiedades
}
```

**Impacto**: Las asignaciones de pestaña y grupo se establecen durante la fase inicial de análisis del modelo.

### 6. Restricciones de edición Enterprise

La visibilidad de pestañas se controla mediante restricciones de edición:

```typescript
// En getVisibleTabs.ts
const isTabVisible =
  localModel?.edition === 'Enterprise' ||
  entity !== 'Models.Entity' ||
  tab?.name !== 'actions'
```

**Impacto**: Ciertas pestañas (como 'actions') solo son visibles en la edición Enterprise o para entidades específicas.

### 7. Procesamiento de valores de formulario

El sistema de valores de formulario filtra los atributos de referencia inversa pero preserva la información de pestaña y grupo:

```typescript
// En useFormValues.ts
attributes
  .filter((attribute) => attribute.incomingReferenceAttributeName === undefined)
  .forEach((attribute) => {
    // Procesar valores de formulario respetando el contexto de pestaña/grupo
  })
```

**Impacto**: Los valores de formulario se procesan independientemente de la visibilidad de pestaña/grupo, pero el renderizado respeta estas asignaciones.

### 8. Filtrado de listas de entidades

Cuando EntityListContainer se usa dentro de contextos de pestaña/grupo, respeta el filtrado:

```typescript
// En EntityViewFieldGroup.tsx - Referencias inversas
<EntityListContainer
  filterAttribute={attribute.name}
  filterValue={entityId}
  entitySchemaName={attribute.entityName}
/>
```

**Impacto**: Las listas de entidades dentro de pestañas/grupos se filtran correctamente y solo muestran datos relevantes.

## Casos de uso comunes

### 1. Entidad simple con campos básicos

```typescript
// Todos los atributos sin pestaña/grupo
// Se renderiza como un formulario simple
```

### 2. Entidad compleja con pestañas

```typescript
// Atributos distribuidos en múltiples pestañas
// Crea una interfaz con pestañas para mejor organización
```

### 3. Entidad solo con grupos

```typescript
// Atributos organizados en grupos sin pestañas
// Proporciona separación visual sin complejidad de pestañas
```

### 4. Organización jerárquica completa

```typescript
// Atributos organizados en grupos dentro de pestañas
// Máxima organización para entidades complejas
```

## Manejo de errores

- Si los valores de `tab` o `group` son `null` o `undefined`, el atributo se trata como sin asignación de pestaña/grupo
- Los IDs de pestaña o grupo inválidos resultan en que el atributo no se muestre
- Las claves de traducción faltantes recurren a los valores de ID sin procesar

## Consideraciones de rendimiento

- La visibilidad de pestañas y grupos se calcula en cada renderizado
- Considere cachear los cálculos de visibilidad para entidades grandes
- El filtrado de grupos incluye verificación de variantes que puede afectar el rendimiento 