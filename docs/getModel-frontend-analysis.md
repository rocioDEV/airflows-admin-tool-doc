# Análisis técnico: Carga de datos getModel en el frontend admin-tool

## Resumen ejecutivo

Este documento describe el flujo completo de carga y almacenamiento de datos del modelo en el frontend de la herramienta de administración de Airflows. El sistema utiliza React Query para el cacheo, Context API para el estado global, y procesamiento personalizado para transformar los datos del backend en estructuras optimizadas para el frontend.

## Arquitectura general

### Componentes principales

1. **App.tsx** - Punto de entrada de la aplicación
2. **Bootstrap.tsx** - Inicialización de componentes base
3. **AllModels.tsx** - Componente que ejecuta la carga de modelos
4. **useFetchModels.ts** - Hook principal que orquesta las consultas
5. **parseModels.ts** - Procesador de datos del modelo
6. **useAppStateContext.tsx** - Contexto global de estado

## Flujo de carga de datos

### 1. Punto de entrada (App.tsx)

```typescript
// Línea 82 en src/App.tsx
<Bootstrap />
```

La aplicación inicia renderizando el componente `Bootstrap` que contiene la lógica de inicialización.

### 2. Componente Bootstrap (src/components/bootstrap/Bootstrap.tsx)

```typescript
// Líneas 41-50
export const Bootstrap = () => {
  useRefreshToken()
  
  return (
    <>
      <Personalization />
      <AllModels />  // Línea 47
    </>
  )
}
```

El componente `Bootstrap` renderiza `AllModels` junto con otros componentes de inicialización.

### 3. Componente AllModels (src/components/bootstrap/all-models/AllModels.tsx)

```typescript
export const AllModels = () => {
  useFetchModels()  // Línea 4
  return null
}
```

Componente minimalista que ejecuta el hook `useFetchModels` para cargar los datos.

### 4. Hook useFetchModels (src/components/bootstrap/all-models/useFetchModels.ts)

Este hook es el orquestador principal que gestiona múltiples consultas:

#### Consultas React Query

```typescript
// Consulta principal getModel (líneas 32-36)
const { data: model } = useQuery({
  queryKey: isAuthenticated ? GET_MODEL : GET_MODEL_ANONYMOUS,
  queryFn: () => getModel({ isMock: state.isMock }),
  enabled: !!accessToken || anonymousLogin,
})

// Consulta getModelData (líneas 38-42)
const { data: allModelData } = useQuery({
  queryKey: isAuthenticated ? GET_MODEL_DATA : GET_MODEL_DATA_ANONYMOUS,
  queryFn: () => getModelData({ isMock: state.isMock }),
  enabled,
})

// Consulta getModelsEntityIdList (líneas 44-48)
const { data: modelsIds } = useQuery({
  queryKey: GET_MODEL_ENTITY_ID,
  queryFn: () => getModelsEntityIdList(),
  enabled: !!accessToken,
})

// Consulta getModelsErrorMessageList (líneas 54-58)
const { data: errorMessagesList } = useQuery({
  queryKey: GET_MODEL_ERROR_MESSAGE_LIST,
  queryFn: getModelsErrorMessageList,
  enabled,
})
```

#### Procesamiento de datos (líneas 82-108)

```typescript
useEffect(() => {
  if (!model || !allModelData || !modelHasEntities(model)) {
    return
  }
  
  // Parseo de recursos i18n
  const resources = parseI18NResources({
    model: model,
    allModelData,
  })
  
  // Carga de traducciones
  Object.keys(resources).forEach((language) => {
    loadTranslations(language, resources[language])
  })
  
  // Parseo principal de modelos
  const parsed = parseModels({
    model: model,
    allModelData,
    contextState: stateRef.current,
    logout,
  })
  
  // Actualización del estado de la aplicación
  if (parsed) {
    setAppContextStateRef.current({
      ...parsed,
      errorMessagesList: errorMessagesList?.result,
    })
  }
}, [model, allModelData, errorMessagesList, logout])
```

### 5. Función getModel (src/data/queries/getModel.ts)

```typescript
export async function getModel({
  isMock,
}: {
  isMock?: boolean
}): Promise<DBParsedModel | ErrorModel | undefined> {
  if (isMock) {
    return getModelMock()
  }

  const vars: GetModelQueryVariables = {}

  const data = await getGraphqlClient().request<GetModelQuery, GetModelQueryVariables>(
    GetModelDocument,
    vars,
    { ...authorizationHeaderHelper() },
  )

  if (data.model) {
    if (typeof data.model === 'string') {
      return JSON.parse(data.model)
    }
    return data.model
  }
  return undefined
}
```

#### Consulta GraphQL (src/data/queries/queriesGraphql/getModel.graphql)

```graphql
query GetModel {
  model: getModel
}
```

### 6. Cliente GraphQL (src/data/graphqlClient.ts)

```typescript
export const getGraphqlClient = () => {
  if (!client) {
    const { baseUrl } = getAppConfig()
    client = new GraphQLClient(baseUrl + '/graphql', {
      fetch: customFetch,
    })
  }
  return client
}
```

#### Configuración de URL (src/components/app-config/getAppConfig.ts)

```typescript
const defaultConfig: Result = {
  baseUrl: origin.includes('localhost') ? 'https://localhost:8443' : origin,
}
```

## Almacenamiento de datos

### 1. Contexto de estado de la aplicación (src/contexts/useAppStateContext.tsx)

```typescript
export type AppState = {
  baseUrl: string
  socket?: WebSocket
  disableHeader: boolean
  disableMenu: boolean
  disableLogViewer: boolean
  activityIndicatorVisible: boolean
  protocol?: string
  hostname?: string
  username?: string
  supportedLanguages: string[]
  themeCustomization: ThemeCustomization
  model?: DBParsedModel           // Modelo parseado desde getModel
  localModel?: LocalModel         // Modelo procesado para frontend
  variables?: GetAllDataQuery['variables']
  errorMessages?: (ErrorMessage | null)[]
  errorMessagesList?: GetModelsErrorMessageListQuery['result']
  rawDBModel?: RawDBModel         // Datos raw del modelo (respaldo)
  rawAllModelData?: GetAllDataQuery // Datos raw de personalización
  isMock: boolean
}
```

### 2. Cache de React Query

- **Claves de consulta**: 
  - `GET_MODEL` (autenticado)
  - `GET_MODEL_ANONYMOUS` (anónimo)
- **Ubicación**: Cache en memoria de React Query
- **Propósito**: Cacheo y refetch automático
- **Archivos**: `src/data/queryKeys.ts` (línea 2)

### 3. Estructura LocalModel (src/data/mocks/localModel.ts)

Datos procesados y estructurados para uso del frontend:

```typescript
export const localModel: LocalModel = {
  entities: {},      // Entidades procesadas
  applications: {},  // Aplicaciones
}
```

### 4. Selectores globales (src/contexts/useAppStateContext.tsx)

```typescript
const selectors = useMemo(
  () =>
    createSelectorsGlobal({
      rawDBModel,
      rawAllModelData,
      errorMessagesList: state.errorMessagesList,
    }),
  [rawAllModelData, rawDBModel, state.errorMessagesList],
)

globalSelectorsHolder.value = selectors
```

## Procesamiento de datos

### Función parseModels (src/components/bootstrap/all-models/parseModels.ts)

Esta función transforma los datos raw del backend en estructuras optimizadas para el frontend:

#### 1. Entidades externas (líneas 44-55)

```typescript
if (allModelData.externalEntities) {
  localModel.entities = {
    ...localModel.entities,
    ...processExternalEntities({
      externalEntities: allModelData.externalEntities,
      dbModelExternalEntities: model.externalEntities,
      isSuper: model.super,
    }),
  }
}
```

#### 2. Aplicaciones (líneas 58-73)

```typescript
if (allModelData.applications) {
  allModelData.applications.forEach((application) => {
    if (!application?.name) return
    
    if (!localModel.applications) {
      localModel.applications = {}
    }

    localModel.applications[application.name] = {
      order: application.order,
      color: application.color,
    }
  })
}
```

#### 3. Entidades principales (líneas 78-169)

```typescript
allModelData.entities?.forEach((entity) => {
  if (!entity) return
  
  const { attributes, tabs, groups, questions } = buildEntityData(entity)
  
  // MODELO LOCAL
  const entityName = entity.schema + '.' + entity.name
  localModel.entities[entityName] = {
    icon: entity.icon,
    menu: entity.menu,
    exportToCSVEnabled: entity.exportToCSVEnabled,
    defaultOrderAttribute: entity.defaultOrderAttribute,
    defaultOrderDesc: entity.defaultOrderDesc,
    defaultOrderNullsLast: entity.defaultOrderNullsLast,
    defaultValues: entity.defaultValues,
    language: entity.language,
    order: entity.order,
    attributes: attributes,
    tabs: tabs,
    groups: groups,
    questions: questions,
  }
  
  // Procesamiento de referencias directas e inversas...
})
```

#### 4. Referencias (líneas 176-191)

```typescript
entityArray.forEach((entity) => {
  if (entity.references !== null) {
    Object.values(entity.references).forEach((reference) => {
      reference.referenceAttributeName =
        reference.referencedKey.entityName.split('.')[1] +
        'Via' +
        reference.name.substring(0, 1).toUpperCase() +
        reference.name.substring(1)
      reference.incomingReferenceAttributeName =
        reference.entityName.split('.')[1] +
        'ListVia' +
        reference.name.substring(0, 1).toUpperCase() +
        reference.name.substring(1)
      reference.referenceAttributeSchema = reference.referencedKey.schema
    })
  }
})
```

## Patrones de acceso a datos

### 1. Hook useAppStateContext

```typescript
const { state } = useAppStateContext()
const model = state.model
const localModel = state.localModel
```

### 2. Selectores globales

```typescript
const selectors = getGlobalSelectors()
// Métodos de acceso convenientes a los datos del modelo
```

### 3. Acceso directo al estado

```typescript
const { state } = useAppStateContext()
// Acceso a campos específicos del modelo
```

## Configuración y URLs

### Configuración de aplicación (src/components/app-config/getAppConfig.ts)

```typescript
const defaultConfig: Result = {
  baseUrl: origin.includes('localhost') ? 'https://localhost:8443' : origin,
}
```

### URLs de GraphQL

- **Desarrollo local**: `https://localhost:8443/graphql`
- **Producción**: `{origin}/graphql`

## Manejo de errores

### 1. Cliente GraphQL personalizado (src/data/graphqlClient.ts)

```typescript
const customFetch: typeof fetch = async (input, init = {}) => {
  // Manejo de errores de sesión
  const errMsg = data?.errors?.[0]?.message
  if (
    errMsg === 'SessionTimeout' ||
    (errMsg?.startsWith('ERROR: role "') && errMsg.endsWith('" does not exist'))
  ) {
    logout()
  }
  // Otros errores...
}
```

### 2. Componente Bootstrap (src/components/bootstrap/Bootstrap.tsx)

```typescript
const originalFetch = window.fetch
window.fetch = async (...args) => {
  // Interceptación global de fetch para manejo de errores
  const conditionTimeout = errorMessage.message === 'SessionTimeout'
  const conditionErrorRole =
    errorMessage.message.startsWith('ERROR: role "') &&
    errorMessage.message.endsWith('" does not exist')
  const permissionDenied = errorMessage.message.includes('ERROR: permission denied')
  
  if (conditionTimeout || conditionErrorRole || permissionDenied) {
    logout()
  }
}
```

## Estados y condiciones

### Habilitación de consultas

```typescript
const enabled = !!accessToken || anonymousLogin
const isAuthenticated = !!accessToken
```

### Verificación de entidades

```typescript
function modelHasEntities(
  model: DBParsedModel | ErrorModel | (DBParsedModel & ErrorModel) | undefined,
): model is DBParsedModel | (DBParsedModel & ErrorModel) {
  return Boolean(model && 'entities' in model && !isNil(model.entities))
}
```

## Persistencia de datos

- **Duración**: Los datos persisten en memoria durante la sesión de la aplicación
- **Refresco automático**: React Query maneja el refetch automático
- **Invalidación**: Se produce al hacer login/logout o cuando React Query lo determina necesario
- **Almacenamiento local**: No hay persistencia en localStorage para los datos del modelo

## Consideraciones de rendimiento

1. **Cache de React Query**: Evita consultas innecesarias
2. **Procesamiento único**: Los datos se procesan una sola vez en `parseModels`
3. **Selectores memoizados**: Los selectores globales se recalculan solo cuando cambian las dependencias
4. **Referencias optimizadas**: Las referencias se procesan y enlazan eficientemente

## Conclusiones

El sistema de carga de datos getModel en el frontend está bien estructurado con separación clara de responsabilidades:

- **React Query** maneja el cacheo y sincronización de datos
- **Context API** proporciona estado global accesible
- **Procesamiento personalizado** optimiza los datos para el frontend
- **Manejo de errores robusto** garantiza la estabilidad de la aplicación

La arquitectura permite escalabilidad y mantenimiento eficiente del código.

