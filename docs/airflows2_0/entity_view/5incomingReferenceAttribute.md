---
sidebar_position: 5
---

# incomingReferenceAttribute

## Descripción general

El `incomingReferenceAttribute` es un atributo de campo utilizado en el componente EntityView para manejar relaciones inversas entre entidades. Define el nombre de un atributo en la entidad referenciada que apunta de vuelta a la entidad actual, creando una relación bidireccional.

## Definición de tipo

```typescript
incomingReferenceAttributeName: string
```

## Propósito

Este atributo se utiliza para establecer referencias inversas, permitiendo que el EntityView muestre entidades relacionadas que referencian a la entidad actual. Es particularmente útil para:

- Mostrar listas de entidades relacionadas
- Crear relaciones bidireccionales
- Mostrar entidades hijas que referencian a una entidad padre

## Uso

### En la definición de atributo

```typescript
const attribute: DBAttributeDefinition = {
  name: 'entidadPadre',
  type: 'reference',
  entityName: 'Models.EntidadPadre',
  incomingReferenceAttributeName: 'ListaEntidadesHijasViaPadre',
  // ... otras propiedades
}
```

### En el renderizado de EntityView

El componente EntityView utiliza este atributo para:

1. **Determinar si es una referencia inversa**: Verifica si `incomingReferenceAttributeName` está definido
2. **Filtrar entidades relacionadas**: Utiliza el nombre del atributo para filtrar entidades que referencian a la entidad actual
3. **Mostrar listas de entidades**: Renderiza un `EntityListContainer` con los resultados filtrados

### Ejemplo de implementación

```typescript
// En EntityViewFieldGroup.tsx
const isInverse = isInverseReference(
  attribute,
  entityState,
  entityLocalModel,
  model,
  tab ?? undefined,
  group ?? undefined,
)

if (isInverse) {
  return (
    <Grid
      id={'inverse-' + attribute.incomingReferenceAttributeName}
      key={attribute.incomingReferenceAttributeName}
    >
      <EntityListContainer
        filterAttribute={attribute.name}
        filterValue={entityId}
        entitySchemaName={attribute.entityName}
      />
    </Grid>
  )
}
```

## Funciones relacionadas

### isInverseReference()

Ubicada en `src/components/entity-view/util/isInverseReference.ts`, esta función determina si un atributo representa una referencia inversa verificando:

1. Si `incomingReferenceAttributeName` está definido
2. Si el atributo coincide con la pestaña y grupo actuales
3. Si la edición del modelo es 'Enterprise' o el atributo no es un tipo de entidad restringido

### Uso en valores de formulario

En `useFormValues.ts`, este atributo se utiliza para filtrar atributos de referencia inversa al construir valores de formulario:

```typescript
.filter((attribute) => attribute.incomingReferenceAttributeName === undefined)
```

## Esquema de base de datos

En el modelo de base de datos (`GetModelType.ts`), este campo se define como:

```typescript
export interface DBEntityReferenceDefinition {
  // ... otras propiedades
  incomingReferenceAttributeName?: string
}
```

## Patrones comunes

### 1. Relaciones padre-hijo

```typescript
// Atributo de entidad padre
{
  name: 'hijos',
  entityName: 'Models.EntidadHija',
  incomingReferenceAttributeName: 'ListaEntidadesHijasViaPadre'
}

// La entidad hija tendría
{
  name: 'padre',
  entityName: 'Models.EntidadPadre',
  referenceAttributeName: 'padreId'
}
```

### 2. Relaciones uno-a-muchos

```typescript
// Entidad principal
{
  name: 'elementosRelacionados',
  entityName: 'Models.ElementoRelacionado',
  incomingReferenceAttributeName: 'ListaElementosRelacionadosViaEntidadPrincipal'
}
```

## Restricciones

Los siguientes tipos de entidades están excluidos del procesamiento de referencias inversas:

- `Models.ExternalEntityPermission`
- `Models.CustomTriggerPermission`
- `Models.RowLevelPermission`
- `Models.EntityAttributePermission`
- `Models.NetworkPermission`
- `Models.SchemaPermission`

## Comportamiento de la UI

Cuando se detecta una referencia inversa:

1. El campo se renderiza como un `EntityListContainer`
2. El contenedor se filtra por el ID de la entidad actual
3. La lista muestra entidades que referencian a la entidad actual
4. El campo respeta la configuración de visibilidad de pestañas y grupos
5. La reserva de espacio se maneja basándose en la selección de variantes

## Manejo de errores

- Si `incomingReferenceAttributeName` es indefinido, el atributo se trata como un campo regular
- Si el atributo referenciado no existe en el modelo local, la referencia inversa no se renderiza
- Los fallos en la coincidencia de pestañas y grupos impiden que se muestre la referencia inversa 