# Documentación técnica de additionalFilter

## Resumen

El `additionalFilter` es una característica poderosa en la herramienta de administración de Airflows que permite **filtrado dinámico y contextual** para atributos de referencia (campos de autocompletado). Permite restringir las opciones disponibles en campos de dropdown/autocompletado basándose en los valores de otros campos en el formulario o entidad actual.

## Arquitectura

### Componentes principales

1. **Configuración del modelo** - Define el filtro en referencias de entidad
2. **Resolución de parámetros** - Procesa parámetros dinámicos en cadenas de filtro
3. **Construcción de consultas GraphQL** - Aplica filtros a consultas del backend
4. **Integración de UI** - Conecta filtros a componentes de autocompletado

## Configuración

### Definición del modelo

El `additionalFilter` se configura en los `directReferences` del modelo de entidad:

```typescript
// En la configuración del modelo de entidad
directReference: {
  name: "assignedUser",
  additionalFilter: "department: {EQ: $currentDepartment}, status: {EQ: 'active'}",
  additionalAttributes: ["department", "role"]
}
```

### Sintaxis de parámetros

Los parámetros usan el prefijo `$` y soportan:
- Referencias simples de campo: `$container`
- Rutas de objetos anidados: `$user.department.name`
- Expresiones complejas: `$currentUser.id`

### Formato de filtro GraphQL

La cadena de filtro sigue la sintaxis de cláusula where de GraphQL:
```graphql
fieldName: {OPERATOR: value}
```

Operadores soportados:
- `EQ` - Igual
- `NE` - No igual
- `IN` - En lista
- `LIKE` - Coincidencia de patrón
- `IS_NULL` - Verificación de nulo

## Flujo de datos

### 1. Carga y análisis del modelo

**Archivo:** `src/components/bootstrap/all-models/parseModels.ts`

```typescript
// Líneas 118-119: additionalFilter se extrae de directReferences
...{ additionalFilter: reference.additionalFilter }, // ignore graphql type
```

El `additionalFilter` se carga desde el modelo de base de datos y se almacena en la configuración del modelo local.

### 2. Búsqueda de referencia y extracción de filtro

**Archivo:** `src/components/entity-view/entity-field/DirectReferenceField/useAdditionalAttributeFilter.ts`

```typescript
// Líneas 21-41: Extraer additionalFilter del modelo local
const { entityFromModel, additionalAttributes, additionalFilter } = useMemo(() => {
  const entityFromModel = selectors.model.getInstance(entitySchemaName)
  const entityLocalModel = localModel.entities[entitySchemaName]
  const entityFromDbModel = selectors.db.getInstance(entitySchemaName)

  // Obtener el nombre del atributo de referencia
  const referenceAttributeName = getReferenceAttributeName(entityFromDbModel, attributeName)

  // Obtener additionalFilter del modelo local
  const localAttribute = referenceAttributeName
    ? entityLocalModel?.attributes?.[referenceAttributeName]
    : undefined
  const additionalFilter = (localAttribute as any)?.additionalFilter

  return {
    entityFromModel,
    additionalAttributes,
    additionalFilter,
  }
}, [selectors, attributeName, entitySchemaName])
```

### 3. Extracción de parámetros

**Archivo:** `src/components/entity-view/entity-field/DirectReferenceField/useAdditionalAttributeFilter.ts`

```typescript
// Líneas 43-58: Extraer parámetros usando regex
const paramsInAdditionalFilter = useMemo(() => {
  if (!additionalFilter) return []

  const regex = /\$([_A-Za-z][_0-9A-Za-z.]*)/g
  const params: string[] = []
  let match: RegExpExecArray | null = regex.exec(additionalFilter)

  while (match !== null) {
    params.push(match[1]) // Nombre del parámetro sin '$'
    match = regex.exec(additionalFilter)
  }

  return Array.from(new Set(params))
}, [additionalFilter])
```

### 4. Resolución de parámetros

**Archivo:** `src/components/entity-view/entity-field/DirectReferenceField/util/buildParameterDictionary.ts`

```typescript
// Construir diccionario de parámetros desde valores del formulario
const filterValues: Record<string, unknown> = useMemo(() => {
  return buildParameterDictionary(paramsInAdditionalFilter, entityFromModel, allValues)
}, [paramsInAdditionalFilter, allValues, entityFromModel])
```

La lógica de resolución de parámetros:
1. **Valores directos del formulario** - Parámetros que coinciden con nombres de campos del formulario
2. **Atributos de referencia** - Parámetros de otros campos de referencia
3. **Propiedades anidadas** - Notación de punto para propiedades de objeto

### 5. Procesamiento de filtros

**Archivo:** `src/components/entity-view/entity-field/DirectReferenceField/useAdditionalAttributeFilter.ts`

```typescript
// Líneas 90-126: Reemplazar parámetros con valores reales
const endFilter = useMemo(() => {
  if (!additionalFilter) return undefined

  let filter = additionalFilter
  let hasUnresolvedParams = false

  paramsInAdditionalFilter.forEach((key) => {
    const value = filterValues[key]
    let replacement: string

    if (value == null || value === '') {
      replacement = 'null'
      hasUnresolvedParams = true
    } else if (typeof value === 'string') {
      const escapedValue = value.replace(/'/g, "\\'")
      replacement = `'${escapedValue}'`
    } else {
      replacement = String(value)
    }

    filter = filter.replace(new RegExp(`\\$${key}`, 'g'), replacement)
  })

  // Retornar undefined si los parámetros no pueden ser resueltos
  return hasUnresolvedParams ? undefined : filter
}, [additionalFilter, filterValues, paramsInAdditionalFilter])
```

### 6. Construcción de consulta GraphQL

**Archivo:** `src/components/ui/autocomplete/hooks/useRefreshData.tsx`

```typescript
// Líneas 123-145: Construir cláusula where de GraphQL
let where = ''
if ((searchCriteria && searchAttributeNames.length) || additionalFilter) {
  where = ' where: { '
  const hasSearchCriteria = searchCriteria && searchAttributeNames.length
  
  if (hasSearchCriteria) {
    where += `${searchAttributeNames[0]}: {SEARCH: {query: "${searchCriteria}" config: ${getLanguage(entityLocalModel.language as string)}}} `
  }
  
  if (additionalFilter) {
    if (hasSearchCriteria) {
      where += ', '
    }
    
    // Validar y sanitizar el additionalFilter
    const sanitizedFilter = sanitizeGraphQLFilter(additionalFilter)
    if (sanitizedFilter) {
      where += sanitizedFilter + ' '
    } else {
      console.warn('Invalid additionalFilter detected, skipping:', additionalFilter)
    }
  }
  
  where += ' } '
}
```

### 7. Ejecución de consulta

**Archivo:** `src/components/ui/autocomplete/hooks/useRefreshData.tsx`

```typescript
// Líneas 201-224: Ejecutar consulta GraphQL
const graphQLQueryName = `${entitySchemaName.replace('.', '_')}List`
const query = `query ${graphQLQueryName} {
  result: ${graphQLQueryName}(
    limit: 1000
    ${orderBy}
    ${where}
  ) {
    ${keyAttr?.name},
    ${additionalAttributes?.join(',') || ''}
    ${getLabelAttributesQueryString(entityModel, entityLocalModel, keyAttr?.name)}
  }
}`

const resultRequest = await graphqlQueryRaw<{
  result: EntityItem[]
}>(query)
```

## Integración de componentes

### DirectReferenceField

**Archivo:** `src/components/entity-view/entity-field/DirectReferenceField/DirectReferenceField.tsx`

```typescript
// Líneas 46-50: Usar hook additionalFilter
const { additionalAttributes, additionalFilter } = useAdditionalAttributeFilter({
  attributeName: attribute.name,
  entitySchemaName: entity,
  allValues,
})

// Líneas 74-76: Pasar a AutoComplete
<AutoComplete
  additionalFilter={additionalFilter}
  additionalAttributes={additionalAttributes ?? undefined}
  // ... otras props
/>
```

### Componente AutoComplete

**Archivo:** `src/components/ui/autocomplete/AutoComplete.tsx`

```typescript
// Líneas 53-58: Pasar al hook useRefreshData
const { refreshData } = useRefreshData({
  model: context.model,
  entitySchemaName,
  additionalFilter,
  additionalAttributes,
})
```

## Consideraciones de seguridad

### Prevención de inyección GraphQL

**Archivo:** `src/components/ui/autocomplete/hooks/useRefreshData.tsx`

```typescript
// Líneas 18-52: función sanitizeGraphQLFilter
function sanitizeGraphQLFilter(filter: string): string | null {
  if (!filter || typeof filter !== 'string') {
    return null
  }

  // Remover patrones peligrosos de GraphQL
  const dangerousPatterns = [
    /__typename/gi,
    /query\s*\{/gi,
    /mutation\s*\{/gi,
    /subscription\s*\{/gi,
    /fragment\s+\w+/gi,
    /\.\.\./g, // Operador spread de GraphQL
  ]

  for (const pattern of dangerousPatterns) {
    if (pattern.test(filter)) {
      console.warn('Potentially dangerous GraphQL pattern detected:', pattern, 'in filter:', filter)
      return null
    }
  }

  // Validación básica - asegurar formato válido de filtro GraphQL
  const validFilterPattern = /^[a-zA-Z_][a-zA-Z0-9_]*\s*:\s*\{[^}]*\}$/
  if (!validFilterPattern.test(filter.trim())) {
    console.warn('Invalid GraphQL filter format:', filter)
    return null
  }

  return filter.trim()
}
```

## Manejo de errores y depuración

### Registro de depuración

El sistema incluye registro de depuración completo en puntos clave:

1. **Depuración de búsqueda de referencia** - Muestra si se encuentra directReference
2. **Depuración de resolución de parámetros** - Muestra valores de parámetros siendo resueltos
3. **Procesamiento final de filtros** - Muestra la cadena de filtro procesada
4. **Construcción de consulta GraphQL** - Muestra la consulta completa siendo enviada

### Escenarios de error

1. **Parámetros no resueltos** - El filtro retorna `undefined` para prevenir consultas inválidas
2. **Sintaxis de filtro inválida** - El filtro es rechazado con advertencias en consola
3. **Referencias faltantes** - Fallback elegante a sin filtrado
4. **Intentos de inyección GraphQL** - El filtro es sanitizado o rechazado

## Ejemplos de uso

### Filtrado básico de contenedor

```typescript
// Configuración del modelo
additionalFilter: "container: {EQ: $container}"

// Resultado: Solo muestra elementos del mismo contenedor que la entidad actual
```

### Filtrado complejo de múltiples campos

```typescript
// Configuración del modelo
additionalFilter: "department: {EQ: $user.department}, status: {IN: ['active', 'pending']}, role: {NE: 'admin'}"

// Resultado: Muestra usuarios del mismo departamento, con estado activo/pendiente, excluyendo administradores
```

### Filtrado condicional

```typescript
// Configuración del modelo
additionalFilter: "type: {EQ: $entityType}, owner: {EQ: $currentUser}"

// Resultado: Muestra elementos de tipo específico propiedad del usuario actual
```

## Consideraciones de rendimiento

1. **Memoización** - Todo el procesamiento de filtros está memoizado para prevenir recálculos innecesarios
2. **Caché de parámetros** - Los valores de parámetros se almacenan en caché hasta que cambien los valores del formulario
3. **Optimización de consultas** - Los filtros solo se aplican cuando los parámetros pueden ser resueltos
4. **Evaluación perezosa** - El procesamiento de filtros solo ocurre cuando es necesario

## Solución de problemas

### Problemas comunes

1. **Filtro no aplicado** - Verificar si los parámetros están resueltos en los logs de consola
2. **GraphQL inválido** - Verificar que la sintaxis del filtro coincida con el formato de cláusula where de GraphQL
3. **Referencias faltantes** - Asegurar que el nombre de directReference coincida con el nombre del atributo
4. **Parámetro no encontrado** - Verificar si el nombre del parámetro coincide con los nombres de campos del formulario

### Lista de verificación de depuración

1. Revisar la consola del navegador para logs de depuración
2. Verificar que `additionalFilter` esté definido en el modelo
3. Confirmar que los nombres de parámetros coincidan con los nombres de campos del formulario
4. Validar la sintaxis del filtro GraphQL
5. Verificar si `allValues` contiene los datos esperados

## Mejoras futuras

1. **Seguridad de tipos** - Agregar tipos TypeScript para parámetros de filtro
2. **Validación** - Validación del lado del cliente de la sintaxis del filtro
3. **Caché** - Cachear filtros resueltos para mejor rendimiento
4. **Pruebas** - Agregar pruebas unitarias completas para el procesamiento de filtros
5. **Documentación** - Auto-generar documentación de filtros desde el modelo
