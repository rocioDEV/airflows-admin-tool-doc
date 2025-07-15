---
sidebar_position: 1
---

# Estructura y hooks

- [Estructura y hooks](#estructura-y-hooks)
  - [Introducción](#introducción)
  - [Árbol de componentes](#árbol-de-componentes)
  - [Hooks críticos](#hooks-críticos)
  - [Interacciones clave](#interacciones-clave)


## Introducción
Este documento describe de forma resumida la estructura y los hooks del componente EntityList (un grid de datos complejos utilizado para listar los datos de cualquier tipo de entidad).

## Árbol de componentes

A continuación se muestra la jerarquía funcional que se renderiza cuando se monta `EntityListContainer`.

```text
EntityListContainer
└─ EntityList
   ├─ EntityListToolbar  ← controles globales (modo de vista, CRUD, exportación, filtros rápidos…)
   │   └─ (interno) LeftSide / ToolbarSearch / ToolbarActions / …
   ├─ Collapse (opcional)
   │   └─ EntityFilters  ← filtros básicos configurables por el usuario
   ├─ Grid
   │   ├─ Paper
   │   │   └─ Box
   │   │       └─ EntityBody  ← representación principal de datos
   │   │            ├─ EntityTableViewer (vista tabla, por defecto)
   │   │            ├─ GalleryViewer (si `galleryEnabled`)
   │   │            ├─ MapViewer (si `mapEnabled`)
   │   │            ├─ CalendarViewer (si `calendarEnabled`)
   │   │            └─ DiagramViewer (si `diagramEnabled`)
   │   └─ Questions (panel lateral opcional con preguntas de negocio)
   └─ DeleteDialog  ← confirmación para borrado masivo
```

**Descripción**

- `EntityListContainer` resuelve el nombre de la entidad vía `react-router` y valida la existencia de su modelo antes de delegar a `EntityList`.
- `EntityList` mantiene el 100 % del estado de UI (elementos seleccionados, orden, paginación, barra de filtros, etc.) y orquesta la obtención de datos mediante hooks.
- Los *viewers* se seleccionan dinámicamente en `EntityBody` según las capacidades detectadas por `useAvailableModes`.
- El árbol está pensado para ocupar todo el alto disponible (`flex:1`) y usa `position:sticky` en la toolbar para que los controles permanezcan visibles durante el *scroll*.

## Hooks críticos

Los siguientes hooks encapsulan la mayor parte de la lógica de negocio de `EntityList`.

| Hook | Propósito | Salida principal |
|------|-----------|------------------|
| `useAvailableModes({ entitySchemaName })` | Determina si la entidad admite vistas especiales (diagrama, galería, mapa, calendario). | `{ diagramEnabled, galleryEnabled, mapEnabled, calendarEnabled, galleryAttribute, mapAttributeNames, calendarAttributes }` |
| `useFetchListData(params)` | Construye dinámicamente la consulta GraphQL y recupera los datos. | `{ data, attributes, hasMoreData, loading, refresh, rawFetchData }` |
| `useSelectManager(data)` | Gestiona la selección de filas mediante un `Set<number>`. | `{ selected, addSelected, removeSelected, selectAll, unselectAll }` |
| `useHeaderSort()` | Mantiene arrays `orderBy` y `orderDirection`; alterna asc → desc → sin orden. | `{ handleSortClick, orderBy, orderDirection }` |
| `useBasicFilterUrl()` y `useSearchValueUrl()` | Persisten filtros y búsqueda en la URL usando `nuqs`. | `{ basicFilters, setBasicFilters }` / `{ searchValue, setSearchValue }` |
| `useDownloadManager({ entitySchemaName, selected, rawFetchData, attributes })` | Exporta datos: descarga directa y exportación a Excel. | `{ handleDownloadClick, handleExportToCSVClick }` |
| `useAttributeActionHandler({ endCallback? })` | Ejecuta acciones basadas en URLs de atributos y navega según corresponda. | `handle(event, baseUrl)` |
| `useDownloadExcel` | Convierte datos a un archivo de Excel en cliente. | `downloadExcel({ ... })` |

## Interacciones clave

1. Al montarse `EntityList`, `useFetchListData` recupera los primeros 100 registros.
2. `EntityListToolbar` invoca `useSelectManager` para habilitar acciones de lote (editar, borrar, exportar).
3. Cambios de orden (`useHeaderSort`) o filtros (`useBasicFilterUrl`) disparan `refresh` en el hook de datos.
4. Acciones “Nueva”, “Ver” o “Editar” usan `useGlobalNavigate` para cambiar de ruta sin perder el *stack* de navegación.
5. El botón “Exportar Excel” llama a `useDownloadManager`, que compone los datos visibles y abre la descarga en cliente. 