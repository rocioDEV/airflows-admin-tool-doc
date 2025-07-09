---
sidebar_position: 8
---

# Sistema de variantes

## Descripción general

El sistema de variantes es un mecanismo sofisticado que controla la visibilidad y el comportamiento de los campos basándose en las selecciones de variantes. Permite el renderizado dinámico de formularios donde ciertos campos, pestañas y grupos solo son visibles cuando se seleccionan valores específicos de variantes. Este sistema es particularmente útil para crear formularios configurables que se adaptan a diferentes escenarios de negocio o configuraciones de productos.

## Conceptos principales

### Variantes

Las variantes son valores predefinidos que se pueden seleccionar para controlar la visibilidad de otros campos. Típicamente se almacenan como arrays de strings y representan diferentes opciones o configuraciones.

### Selectores de variantes

Los campos marcados como `variantSelector: true` actúan como controladores que activan cambios en la visibilidad de otros campos. Cuando cambia un campo selector de variantes, actualiza el estado `selectedVariants`, que a su vez afecta la visibilidad de los campos que dependen de esas variantes.

### Dependencias de variantes

Los campos pueden especificar de qué variantes dependen a través de la propiedad `variants`. Un campo solo es visible cuando al menos una de sus variantes dependientes está seleccionada.

## Definiciones de tipos

```typescript
// En el tipo Attribute
variants: string[]
variantSelector: string

// En el tipo EntityState
selectedVariants?: Record<string, string[]>
```

## Cómo funciona el sistema de variantes

### 1. Selección de variantes

Cuando un usuario selecciona un valor en un campo selector de variantes, el sistema actualiza el estado `selectedVariants`:

```typescript
const setSelectedVariants = (attributeName: string, selectedVariants: string[]) => {
  setEntityState((state) => {
    const newSelectedVariants = {
      ...state.selectedVariants,
      [attributeName]: selectedVariants,
    }
    return { ...state, selectedVariants: newSelectedVariants }
  })
}
```

### 2. Verificación de variantes

El sistema utiliza la función `areVariantsSelectedForThisAttribute` para determinar si un campo debe ser visible:

```typescript
function areVariantsSelectedForThisAttribute(
  attributeVariants: string[],
  selectedVariants: Record<string, string[]> | undefined,
) {
  if (!attributeVariants?.length) {
    return true
  }

  return Object.values(selectedVariants ?? {}).some((variant) =>
    attributeVariants.includes('' + variant),
  )
}
```

**Lógica**:
- Si el atributo no tiene variantes (`!attributeVariants?.length`), siempre es visible
- Si el atributo tiene variantes, solo es visible si al menos una de sus variantes está seleccionada
- La función convierte las variantes a strings para la comparación (`'' + variant`)

### 3. Determinación de visibilidad de campos

Los campos solo se renderizan cuando sus variantes están seleccionadas:

```typescript
const variantSelected = areVariantsSelectedForThisAttribute(attributeVariants, selectedVariants)

// El campo solo se renderiza si variantSelected es true
if (variantSelected) {
  return <EntityField />
}
```

## Campos selectores de variantes

### Campos enum como selectores de variantes

Los campos enum pueden actuar como selectores de variantes cuando se establece la propiedad `variantSelector`:

```typescript
// En EnumField.tsx
const onEnumFieldChange = (event: SelectChangeEvent<string>) => {
  const { options } = event.target as HTMLSelectElement
  const currentAttributeValue: string[] = []

  for (let i = 0, l = options.length; i < l; i += 1) {
    if (options[i].selected) {
      currentAttributeValue.push(options[i].value)
    }
  }

  if (attribute.variantSelector) {
    setSelectedVariants(attribute.name, currentAttributeValue)
  }
  field.onChange(currentAttributeValue)
}
```

**Comportamiento**:
- Cuando cambia un campo selector de variantes, actualiza el estado `selectedVariants`
- Esto activa una re-renderización de todos los campos dependientes
- Se pueden seleccionar múltiples valores (para campos array)

## Integración con otros sistemas

### 1. Visibilidad de pestañas

El sistema de variantes afecta la visibilidad de las pestañas a través de la función `getVisibleTabs`:

```typescript
export function getVisibleTabs(
  entityLocalModel: EntityConfig,
  entity: string,
  selectedVariants: string[],
) {
  return (
    entityLocalModel?.tabs &&
    Object.values(entityLocalModel?.tabs)
      .sort((a, b) => (a?.order ?? 0) - (b?.order ?? 0))
      .filter((tab) => {
        const tabsAttributes =
          entityLocalModel.attributes &&
          Object.values(entityLocalModel.attributes).filter(
            (attribute) =>
              tab &&
              attribute?.tab === tab.id &&
              (!attribute?.variants ||
                attribute?.variants?.length === 0 ||
                (selectedVariants &&
                  selectedVariants.some((variant) => attribute?.variants?.includes('' + variant)))),
          )

        const tabHasAttributes = entityLocalModel.attributes && tabsAttributes?.length
        return isTabVisible && tabHasAttributes
      })
  )
}
```

**Lógica**:
- Una pestaña solo es visible si contiene al menos un atributo que es visible
- La visibilidad del atributo se determina por la selección de variantes
- Las pestañas sin atributos dependientes de variantes siempre son visibles

### 2. Visibilidad de grupos

Los grupos también se ven afectados por el sistema de variantes:

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
        (!attribute.variants ||
          attribute.variants?.length === 0 ||
          (selectedVariants &&
            selectedVariants.some((variant) => attribute.variants?.includes('' + variant))))
      )
    }).length > 0
  )
}
```

**Lógica**:
- Un grupo solo es visible si contiene al menos un atributo visible
- La visibilidad del grupo se determina por la visibilidad de sus atributos

### 3. Reserva de espacio

El sistema de variantes afecta la reserva de espacio para referencias inversas:

```typescript
function isSpaceReservedForInverseReference(
  variantSelected: boolean,
  tab: number | undefined,
  entityLocalModel: EntityConfig,
  attribute: DBAttributeDefinition,
  selectedTab: number | undefined,
) {
  return (
    variantSelected &&
    (tab === null ||
      tab === undefined ||
      entityLocalModel?.attributes?.[attribute.incomingReferenceAttributeName]?.tab === selectedTab)
  )
}
```

**Lógica**:
- El espacio solo se reserva para referencias inversas cuando sus variantes están seleccionadas
- Esto previene cambios de diseño cuando cambian las variantes

## Patrones de uso

### 1. Dependencia simple de variantes

```typescript
// Campo que depende de una sola variante
const attribute: DBAttributeDefinition = {
  name: 'dependentField',
  type: 'TEXT',
  variants: ['option1'],
  // ... otras propiedades
}
```

### 2. Dependencias múltiples de variantes

```typescript
// Campo que depende de múltiples variantes
const attribute: DBAttributeDefinition = {
  name: 'dependentField',
  type: 'TEXT',
  variants: ['option1', 'option2', 'option3'],
  // ... otras propiedades
}
```

### 3. Campo selector de variantes

```typescript
// Campo que actúa como selector de variantes
const attribute: DBAttributeDefinition = {
  name: 'productType',
  type: 'ENUM',
  variantSelector: true,
  enumType: {
    schema: 'public',
    name: 'ProductType',
    values: { 'basic': 'Basic', 'premium': 'Premium', 'enterprise': 'Enterprise' }
  },
  // ... otras propiedades
}
```

### 4. Cadena de dependencias compleja

```typescript
// Múltiples campos con variantes interdependientes
const productType: Attribute = {
  name: 'productType',
  type: 'ENUM',
  variantSelector: true,
  enumType: { /* ... */ }
}

const basicFeatures: Attribute = {
  name: 'basicFeatures',
  type: 'TEXT',
  variants: ['basic'],
  // Solo visible cuando 'basic' está seleccionado
}

const premiumFeatures: Attribute = {
  name: 'premiumFeatures',
  type: 'TEXT',
  variants: ['premium', 'enterprise'],
  // Visible cuando 'premium' o 'enterprise' está seleccionado
}
```

## Gestión de estado

### Estructura de EntityState

```typescript
export type EntityState = {
  // ... otras propiedades
  selectedVariants?: Record<string, string[]>
}
```

### Proveedor de contexto

El sistema de variantes se gestiona a través del contexto EntityView:

```typescript
// En EntityViewProvider.tsx
const setSelectedVariants = (attributeName: string, selectedVariants: string[]) => {
  setEntityState((state) => {
    const newSelectedVariants = {
      ...state.selectedVariants,
      [attributeName]: selectedVariants,
    }
    return { ...state, selectedVariants: newSelectedVariants }
  })
}
```

## Consideraciones de rendimiento

### 1. Re-renderizado

- Los cambios en `selectedVariants` activan re-renderizados de todos los campos dependientes
- Considera memorizar cálculos costosos en componentes dependientes de variantes
- Un gran número de campos dependientes de variantes puede afectar el rendimiento

### 2. Actualizaciones de estado

- Cada cambio en el selector de variantes actualiza todo el objeto `selectedVariants`
- Considera agrupar actualizaciones para múltiples cambios de variantes
- La estructura de estado soporta múltiples selectores de variantes simultáneamente

### 3. Operaciones de filtrado

- La verificación de variantes ocurre en cada renderizado para cada campo
- Considera cachear cálculos de visibilidad de variantes para formularios grandes
- La lógica de filtrado es relativamente ligera pero puede acumularse con muchos campos

## Manejo de errores

### 1. Variantes faltantes

- Los campos sin variantes siempre son visibles
- Las referencias de variantes inválidas se ignoran
- El sistema maneja graciosamente arrays de variantes undefined o null

### 2. Conversión de tipos

- Las variantes se convierten a strings para la comparación (`'' + variant`)
- Esto maneja tanto valores de variantes string como numéricos
- Asegura comportamiento de comparación consistente

### 3. Consistencia de estado

- El estado `selectedVariants` se inicializa como un objeto vacío
- Los selectores de variantes faltantes se manejan graciosamente
- El sistema mantiene consistencia incluso con actualizaciones de estado parciales

## Mejores prácticas

### 1. Nomenclatura de variantes

- Usa nombres descriptivos y consistentes para las variantes
- Evita caracteres especiales que puedan causar problemas de comparación
- Considera usar constantes para valores de variantes

### 2. Gestión de dependencias

- Mantén las dependencias de variantes simples y lógicas
- Evita dependencias circulares entre selectores de variantes
- Documenta relaciones complejas de variantes

### 3. Optimización de rendimiento

- Limita el número de campos dependientes de variantes por formulario
- Usa selectores de variantes con moderación para formularios complejos
- Considera carga diferida para secciones fuertemente dependientes de variantes

### 4. Experiencia de usuario

- Proporciona retroalimentación clara cuando las variantes cambian la visibilidad
- Considera mostrar contenido de marcador de posición para campos ocultos
- Mantén consistencia del estado del formulario a través de cambios de variantes 