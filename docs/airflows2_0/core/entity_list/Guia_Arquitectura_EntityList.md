# Guía arquitectónica del componente EntityList

## Resumen

El componente EntityList es el componente central para mostrar y gestionar datos de entidades en la herramienta de administración. Proporciona una interfaz completa para visualizar, buscar, filtrar y manipular registros de entidades con soporte para múltiples modos de vista y características avanzadas.

## Jerarquía de componentes

```
EntityListContainer
└── EntityList
    ├── EntityListToolbar
    │   ├── LeftSide
    │   ├── ToolbarActions
    │   │   └── ToolbarSearch
    │   └── ToolbarSelectionActions
    ├── EntityFilters (colapsable)
    └── EntityBody
        └── [Varios modos de vista: tabla, galería, calendario, mapa, diagrama]
```

## Componentes principales

### 1. EntityListContainer (`/src/components/entity-list/EntityListContainer.tsx`)

**Propósito**: Componente contenedor que maneja la validación del esquema de entidad y el enrutamiento.

**Responsabilidades clave**:
- Valida el nombre del esquema de entidad desde parámetros de URL o props
- Verifica si el modelo de entidad existe en el sistema
- Renderiza estado de error si la entidad no se encuentra
- Pasa props validadas a EntityList

**Props**:
- `entitySchemaName`: Identificador de entidad (ej., "Models.Application")
- `filterAttribute`: Atributo opcional para filtrar
- `filterValue`: Valor para el atributo de filtro
- `editionDisabled`: Deshabilita operaciones de edición
- `disabled`: Deshabilita todas las operaciones
- `stickyTop`: Desplazamiento de posicionamiento fijo
- `isFromEntityView`: Bandera de contexto para integración con vista de entidad

### 2. EntityList (`/src/components/entity-list/EntityList.tsx`)

**Propósito**: Componente principal que orquesta toda la funcionalidad de la lista de entidades.

**Responsabilidades clave**:
- Gestiona el estado para filtros, ordenamiento y paginación
- Coordina entre la barra de herramientas, filtros y visualización de datos
- Maneja operaciones de entidad (ver, editar, eliminar, crear)
- Gestiona el estado de selección
- Se integra con varios modos de vista

**Gestión de estado**:
- Utiliza múltiples hooks personalizados para diferentes preocupaciones
- Sincronización de estado URL para búsqueda, filtros y modos de vista
- Estado local para interacciones de UI (visibilidad de filtros, selección)

**Hooks clave utilizados**:
- `useSearchValueUrl`: Gestiona el término de búsqueda en la URL
- `useBasicFilterUrl`: Gestiona filtros básicos en la URL
- `useViewModeStateUrl`: Gestiona el modo de vista en la URL
- `useHeaderSort`: Gestiona el estado de ordenamiento
- `useFetchListData`: Maneja la obtención de datos y paginación
- `useSelectManager`: Gestiona la selección de elementos
- `useAvailableModes`: Determina los modos de vista disponibles

### 3. EntityListToolbar (`/src/components/entity-list/components/EntityListToolbar.tsx`)

**Propósito**: Barra de herramientas superior que proporciona acciones y controles.

**Responsabilidades clave**:
- Muestra información de entidad y conteo de selección
- Proporciona botones de acción (ver, editar, eliminar, crear)
- Muestra selectores de modo de vista
- Integra funcionalidad de búsqueda
- Maneja controles de filtro

**Renderizado condicional**:
- Muestra diferentes acciones basadas en el estado de selección
- Habilita/deshabilita características basadas en permisos de usuario
- Se adapta a los modos de vista disponibles

### 4. ToolbarSearch (`/src/components/ui/table/ToolbarSearch.tsx`)

**Propósito**: Componente de entrada de búsqueda con sincronización de URL con debounce.

**Características clave**:
- Debounce de 500ms para prevenir llamadas excesivas a la API
- Sincronización de estado URL usando `useSearchValueUrl`
- Funcionalidad de botón de limpiar
- Soporte para estado deshabilitado
- Tooltip con instrucciones de búsqueda

**Comportamiento de búsqueda**:
- Actualiza el parámetro de consulta URL `search`
- Activa la actualización de datos a través de `useFetchListData`
- Mantiene el estado de búsqueda a través de la navegación

## Arquitectura de flujo de datos

### Flujo de funcionalidad de búsqueda

```
Usuario escribe en búsqueda → ToolbarSearch → useSearchValueUrl → Actualización URL → 
useFetchListData → rawFetchData → buildWhere → buildSearchCriteria → 
Consulta GraphQL → Búsqueda en backend → Resultados mostrados
```

### Flujo de búsqueda detallado

1. **Entrada del usuario** (`ToolbarSearch.tsx`):
   - El usuario escribe en el campo de búsqueda
   - El estado local se actualiza inmediatamente para responsividad de UI
   - El efecto con debounce (500ms) actualiza el estado URL

2. **Gestión de estado URL** (`useSearchValueUrl.ts`):
   - Utiliza la librería `nuqs` para el estado de consulta URL
   - Mantiene el parámetro de consulta `search`
   - Proporciona `searchValue` y `setSearchValue`

3. **Obtención de datos** (`useFetchListData.ts`):
   - Monitorea cambios en `searchValue`
   - Activa `refresh()` cuando cambia la búsqueda
   - Llama a `rawFetchData` con parámetros de búsqueda

4. **Construcción de consulta** (`rawFetchData.ts`):
   - Llama a `buildWhere` con criterios de búsqueda
   - Construye consulta GraphQL con cláusula WHERE
   - Ejecuta consulta vía `graphqlQueryRaw`

5. **Procesamiento de criterios de búsqueda** (`buildWhere.ts` + `buildSearchCriteria.ts`):
   - Procesa entrada de búsqueda en bruto
   - Aplica búsqueda de texto completo a campos configurados
   - Maneja configuración de búsqueda específica por idioma

## Detalles de implementación de búsqueda

### Procesamiento de criterios de búsqueda

La funcionalidad de búsqueda utiliza búsqueda de texto completo de PostgreSQL con el siguiente procesamiento:

1. **Sanitización de entrada** (`buildSearchCriteria.ts`):
   ```typescript
   const raw = rawInput.replace(/"/g, "'").replace(/\\/g, '').trim()
   ```

2. **Procesamiento de tokens**:
   - Divide la entrada por espacios en blanco
   - Añade sufijo `:*` a tokens no operadores para coincidencia de prefijo
   - Preserva operadores de búsqueda de texto completo de PostgreSQL

3. **Construcción de consulta**:
   - Une tokens con `&` para operaciones AND
   - Preserva paréntesis y operadores
   - Ejemplo: "hello world" → "hello:* & world:*"

### Configuración de campo de búsqueda

La búsqueda se habilita por entidad a través de la propiedad `textSearch` en las claves de entidad:

```typescript
// useSearchEnabled.ts
return Object.values(entityModel.keys).some((key) => key.textSearch)
```

### Soporte de idiomas

La búsqueda soporta múltiples idiomas a través de configuraciones de búsqueda de texto de PostgreSQL:

- Mapea códigos de idioma a configuraciones de PostgreSQL
- Por defecto usa configuración 'ENGLISH'
- Soporta 15+ idiomas incluyendo español, francés, alemán, etc.

### Estructura de consulta GraphQL

La búsqueda se integra en la consulta principal de lista de entidades:

```graphql
query EntityList {
  result: entityList(
    offset: 0
    limit: 125
    where: {AND: [
      {field: {SEARCH: {query: "processed:search:terms", config: "SPANISH"}}}
    ]}
  ) {
    id
    attribute1
    attribute2
    # ... otros atributos
  }
}
```

## Patrones de gestión de estado

### Sincronización de estado URL

El componente utiliza estado URL para:
- Términos de búsqueda (parámetro de consulta `search`)
- Filtros básicos (parámetro de consulta `filters`)
- Modo de vista (parámetro de consulta `mode`)
- Ordenamiento (parámetros de consulta `orderBy`, `orderDirection`)

### Gestión de estado local

El estado local se utiliza para:
- Interacciones de UI (visibilidad del panel de filtros)
- Estado de selección (elementos seleccionados)
- Estados de carga
- Desplazamiento de paginación

### Estrategia de obtención de datos

- **Paginación**: 125 elementos por página (por encima de 100 para evitar saltos en la visualización)
- **Debouncing**: 500ms para entrada de búsqueda
- **Caché**: Resultados cacheados en el estado del componente
- **Estados de carga**: Mínimo 500ms de visualización de carga para UX

## Puntos de integración

### Modos de vista

El componente soporta múltiples modos de vista:
- **Tabla**: Vista tabular estándar
- **Galería**: Vista basada en tarjetas con soporte de imagen
- **Calendario**: Vista basada en fechas
- **Mapa**: Vista geográfica
- **Diagrama**: Vista basada en grafos

### Sistema de permisos

Las acciones se controlan por:
- Rol de usuario (bandera `isSuper`)
- Privilegios a nivel de entidad (`INSERT`, `EXPORT`, etc.)
- Banderas de características por entidad

### Operaciones de entidad

Operaciones soportadas:
- **Ver**: Navegar a detalle de entidad
- **Editar**: Navegar a formulario de edición de entidad
- **Eliminar**: Eliminación masiva de elementos seleccionados
- **Crear**: Navegar a formulario de creación de entidad
- **Exportar**: Descargar datos como CSV/Excel

## Consideraciones de rendimiento

### Estrategias de optimización

1. **Búsqueda con debounce**: Previene llamadas excesivas a la API
2. **Paginación**: Limita transferencia de datos
3. **Carga selectiva de atributos**: Solo carga atributos requeridos
4. **Cálculos memoizados**: Previene re-renderizados innecesarios
5. **Gestión de estado de carga**: Tiempo mínimo de visualización para UX suave

### Gestión de memoria

- Estado del componente limpiado al desmontar
- Estado URL persiste a través de la navegación
- Conjuntos de datos grandes manejados a través de paginación

## Manejo de errores

### Errores de red
- Sistema de mensajes de error global
- Degradación elegante en fallos de API
- Mecanismos de reintento para solicitudes fallidas

### Errores de validación
- Validación de esquema de entidad
- Verificaciones de permisos
- Validación de entrada para términos de búsqueda

## Características de accesibilidad

- Etiquetas ARIA para todos los elementos interactivos
- Soporte de navegación por teclado
- Compatibilidad con lectores de pantalla
- Instrucciones de tooltip para funcionalidad de búsqueda

## Extensibilidad futura

La arquitectura soporta:
- Nuevos modos de vista a través de `useAvailableModes`
- Implementaciones de búsqueda personalizadas
- Tipos de filtro adicionales
- Modelos de permisos mejorados
- Operaciones de entidad personalizadas
