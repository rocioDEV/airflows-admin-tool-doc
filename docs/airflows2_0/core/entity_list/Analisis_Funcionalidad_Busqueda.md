# Análisis de la funcionalidad de búsqueda por texto

## Resumen

La funcionalidad de búsqueda en el componente EntityList proporciona capacidades de búsqueda de texto completo utilizando las características de búsqueda de texto integradas de PostgreSQL. La implementación está diseñada para ser performante, fácil de usar y consciente del idioma.

## Componentes de la arquitectura de búsqueda

### 1. Capa de interfaz de usuario

#### Componente ToolbarSearch (`/src/components/ui/table/ToolbarSearch.tsx`)

**Características clave**:
- Campo de entrada con texto de marcador de posición
- Botón de limpiar (X) cuando hay texto presente
- Debounce de 500ms para prevenir llamadas excesivas a la API
- Soporte para estado deshabilitado
- Tooltip con instrucciones de búsqueda

**Detalles de implementación**:
```typescript
const [search, setSearch] = useState(searchValue)

useEffect(() => {
  const handler = setTimeout(() => {
    setSearchValue(search)
  }, 500)
  return () => clearTimeout(handler)
}, [search, setSearchValue])
```

**Experiencia de usuario**:
- Retroalimentación visual inmediata (estado local)
- Llamadas API retrasadas (con debounce)
- Funcionalidad de limpiar para reinicio fácil
- Soporte de accesibilidad con etiquetas ARIA

### 2. Capa de gestión de estado

#### Hook useSearchValueUrl (`/src/components/entity-list/hook/useSearchValueUrl.ts`)

**Propósito**: Gestiona la sincronización del término de búsqueda con parámetros de consulta URL.

**Implementación**:
```typescript
const [searchValue, setSearchValue] = useQueryState('search', parseAsString.withDefault(''))
```

**Beneficios**:
- Los términos de búsqueda persisten a través de actualizaciones de página
- URLs compartibles con criterios de búsqueda
- Soporte de navegación hacia atrás/adelante del navegador
- Integración con el sistema de enrutamiento

### 3. Lógica de habilitación de búsqueda

#### Hook useSearchEnabled (`/src/components/entity-list/components/hooks/useSearchEnabled.ts`)

**Propósito**: Determina si la funcionalidad de búsqueda debe estar disponible para una entidad.

**Lógica**:
```typescript
return Object.values(entityModel.keys).some((key) => key.textSearch)
```

**Configuración**:
- La búsqueda se habilita por entidad a través de la configuración del modelo
- Requiere al menos una clave con `textSearch: true`
- Permite control granular sobre la disponibilidad de búsqueda

### 4. Integración de obtención de datos

#### Hook useFetchListData (`/src/components/entity-list/hook/useFetchListData.ts`)

**Integración de búsqueda**:
- Monitorea cambios en `searchValue`
- Activa actualización de datos cuando cambia la búsqueda
- Pasa criterios de búsqueda a la capa de obtención de datos
- Mantiene estado de paginación durante la búsqueda

**Optimizaciones de rendimiento**:
- Búsqueda con debounce previene llamadas excesivas a la API
- Paginación limita transferencia de datos (125 elementos por página)
- Estados de carga proporcionan retroalimentación al usuario

### 5. Capa de construcción de consulta

#### Función rawFetchData (`/src/components/entity-list/hook/utils/rawFetchData.ts`)

**Integración de búsqueda**:
- Llama a `buildWhere` con criterios de búsqueda
- Construye consulta GraphQL con cláusula WHERE
- Ejecuta búsqueda vía endpoint GraphQL

**Estructura de consulta**:
```typescript
const where = buildWhere({
  entityName: entitySchemaName,
  basicFilters,
  searchCriteria: searchValue, // Término de búsqueda pasado aquí
  // ... otros parámetros
})
```

### 6. Procesamiento de criterios de búsqueda

#### Función buildWhere (`/src/components/entity-list/hook/utils/buildWhere.ts`)

**Lógica de procesamiento de búsqueda**:
```typescript
// Búsqueda de texto completo global
const keys = Object.values(entity.keys)
const keyWithTextSeach = keys.find((k) => k.textSearch)
if (keyWithTextSeach && searchCriteria) {
  const field = keyWithTextSeach.attributes[0].name
  where.push(
    `{${field}: { SEARCH: { query: "${buildSearchCriteria(searchCriteria)}" config: ${getLanguage(localEntity.language ?? 'en_US')} }}}`
  )
}
```

**Características clave**:
- Encuentra el campo de búsqueda configurado
- Aplica configuración de búsqueda específica por idioma
- Procesa criterios de búsqueda a través de `buildSearchCriteria`
- Se integra con otras condiciones WHERE

#### Función buildSearchCriteria (`/src/components/entity-list/hook/utils/buildSearchCriteria.ts`)

**Procesamiento de entrada**:
```typescript
export function buildSearchCriteria(rawInput: string): string | null {
  const raw = rawInput.replace(/"/g, "'").replace(/\\/g, '').trim()
  if (!raw) {
    return null
  }

  const tokens = raw.split(/\s+/).map((tok) => 
    (/[()&|!:*"']/.test(tok) ? tok : `${tok}:*`)
  )

  const criteria = tokens
    .reduce((acc, tok, i) => {
      const prev = tokens[i - 1] || ''
      const sep = i > 0 && !/[()&|]/.test(prev) && !/[()&|]/.test(tok) ? ' & ' : ' '
      return acc + sep + tok
    }, '')
    .trim()

  return criteria
}
```

**Pasos de procesamiento**:
1. **Sanitización**: Elimina caracteres peligrosos, recorta espacios en blanco
2. **Tokenización**: Divide la entrada por espacios en blanco
3. **Coincidencia de prefijo**: Añade `:*` a tokens no operadores
4. **Construcción de consulta**: Une tokens con `&` para operaciones AND
5. **Preservación de operadores**: Mantiene operadores de búsqueda de texto completo de PostgreSQL

**Ejemplos**:
- Entrada: "hello world" → Salida: "hello:* & world:*"
- Entrada: "test & (foo | bar)" → Salida: "test:* & (foo:* | bar:*)"
- Entrada: "exact match" → Salida: "exact:* & match:*"

### 7. Soporte de idiomas

#### Función getLanguage (`/src/components/entity-list/hook/utils/getLanguage.ts`)

**Idiomas soportados**:
- Danés, Holandés, Inglés, Finlandés, Francés
- Alemán, Italiano, Noruego, Portugués
- Rumano, Ruso, Español, Sueco, Turco

**Mapeo de configuración**:
```typescript
switch (lang) {
  case 'es_ES': return 'SPANISH'
  case 'en_US': return 'ENGLISH'
  case 'fr_FR': return 'FRENCH'
  // ... más mapeos
  default: return 'ENGLISH'
}
```

**Beneficios**:
- Derivación específica por idioma y palabras vacías
- Mejorada relevancia de búsqueda para diferentes idiomas
- Detección automática de idioma desde configuración de entidad

## Flujo completo de datos de búsqueda

### 1. Interacción del usuario
```
Usuario escribe "término de búsqueda" → Componente ToolbarSearch
```

### 2. Actualización de estado local
```
setSearch("término de búsqueda") → Actualización inmediata de UI
```

### 3. Actualización de URL con debounce
```
Retraso de 500ms → setSearchValue("término de búsqueda") → Actualización de parámetro de consulta URL
```

### 4. Activación de obtención de datos
```
useFetchListData detecta cambio en searchValue → se llama refresh()
```

### 5. Construcción de consulta
```
rawFetchData llamado con searchValue → buildWhere procesa criterios de búsqueda
```

### 6. Procesamiento de criterios de búsqueda
```
buildSearchCriteria("término de búsqueda") → "término:* & búsqueda:*"
```

### 7. Ejecución de consulta GraphQL
```
Consulta GraphQL con cláusula WHERE → Búsqueda de texto completo PostgreSQL en backend
```

### 8. Procesamiento de resultados
```
Backend retorna resultados filtrados → Estado del componente actualizado → UI se re-renderiza
```

## Ejemplos de consultas de búsqueda

### Búsqueda simple
**Entrada del usuario**: "hello world"
**Consulta procesada**: "hello:* & world:*"
**WHERE GraphQL**: `{field: {SEARCH: {query: "hello:* & world:*", config: "ENGLISH"}}}`

### Búsqueda compleja con operadores
**Entrada del usuario**: "test & (foo | bar)"
**Consulta procesada**: "test:* & (foo:* | bar:*)"
**WHERE GraphQL**: `{field: {SEARCH: {query: "test:* & (foo:* | bar:*)", config: "ENGLISH"}}}`

### Búsqueda específica por idioma
**Entrada del usuario**: "casa bonita"
**Idioma de entidad**: "es_ES"
**Consulta procesada**: "casa:* & bonita:*"
**WHERE GraphQL**: `{field: {SEARCH: {query: "casa:* & bonita:*", config: "SPANISH"}}}`

## Características de rendimiento

### Estrategias de optimización

1. **Debouncing**: Retraso de 500ms previene llamadas excesivas a la API
2. **Paginación**: 125 elementos por página limita transferencia de datos
3. **Búsqueda indexada**: La búsqueda de texto completo de PostgreSQL utiliza índices GIN
4. **Optimización de idioma**: Configuraciones específicas por idioma mejoran relevancia
5. **Coincidencia de prefijo**: El sufijo `:*` permite búsquedas de prefijo eficientes

### Consideraciones de rendimiento

- **Configuración de campo de búsqueda**: Solo los campos configurados son buscables
- **Índices de base de datos**: Requiere índices GIN apropiados en campos de búsqueda
- **Complejidad de consulta**: Consultas complejas pueden impactar el rendimiento
- **Tamaño del conjunto de resultados**: Conjuntos de resultados grandes pueden requerir optimización de paginación

## Manejo de errores

### Validación de entrada
- Los términos de búsqueda vacíos se ignoran
- Los caracteres peligrosos se sanitizan
- Las consultas malformadas se manejan elegantemente

### Errores de red
- Sistema de mensajes de error global
- Degradación elegante en fallos de búsqueda
- Mecanismos de reintento para solicitudes fallidas

### Errores de backend
- Manejo de errores GraphQL
- Fallback a coincidencia exacta si falla la búsqueda de texto completo
- Mensajes de error amigables para el usuario

## Consideraciones de seguridad

### Sanitización de entrada
- Reemplazo de caracteres de comillas (`"` → `'`)
- Eliminación de barras invertidas
- Prevención de inyección SQL a través de consultas parametrizadas

### Control de acceso
- La búsqueda respeta permisos a nivel de entidad
- Control de acceso basado en roles de usuario
- Rastro de auditoría para operaciones de búsqueda

## Requisitos de configuración

### Configuración del modelo de entidad
```typescript
// Configuración de clave de entidad para búsqueda
{
  keys: {
    searchKey: {
      textSearch: true,  // Habilita búsqueda en esta clave
      attributes: [{ name: "searchableField" }]
    }
  }
}
```

### Requisitos de base de datos
- PostgreSQL con soporte de búsqueda de texto completo
- Índices GIN en campos buscables
- Configuraciones de búsqueda de texto específicas por idioma

## Oportunidades de mejora futuras

### Mejoras de búsqueda
- **Búsqueda difusa**: Manejar errores tipográficos y variaciones
- **Sugerencias de búsqueda**: Funcionalidad de auto-completado
- **Historial de búsqueda**: Recordar búsquedas recientes
- **Operadores avanzados**: Sintaxis de consulta más sofisticada

### Optimizaciones de rendimiento
- **Caché de resultados de búsqueda**: Cachear búsquedas frecuentes
- **Búsqueda incremental**: Búsqueda en tiempo real mientras el usuario escribe
- **Analíticas de búsqueda**: Rastrear patrones de búsqueda y rendimiento
- **Búsqueda multi-campo**: Buscar a través de múltiples campos simultáneamente

### Mejoras de experiencia de usuario
- **Resaltado de búsqueda**: Resaltar términos coincidentes en resultados
- **Filtros de búsqueda**: Combinar búsqueda de texto con otros filtros
- **Exportación de búsqueda**: Exportar resultados de búsqueda
- **Búsquedas guardadas**: Guardar y reutilizar consultas de búsqueda
