# Modelo de datos de Airflows: guía completa

## Resumen ejecutivo

El frontend de Airflows gestiona datos de tres fuentes principales que se combinan para crear un modelo de datos unificado y robusto. Esta arquitectura permite separar la estructura de la base de datos, las configuraciones del frontend y las mejoras en tiempo de ejecución, proporcionando flexibilidad y capacidad de recuperación.

Además, Airflows define un DSL (lenguaje específico de dominio) que describe el modelo persistido (base de datos), permisos, funciones y metadatos de UI. Este DSL es la fuente de verdad del dominio y se refleja en la base de datos y en las configuraciones expuestas por el backend.

## modelo DSL (persistido): componentes y conceptos

### estructura del modelo

El `Model` raíz contiene todos los demás componentes:
- `applications` - Definiciones de aplicaciones
- `entities` - Entidades de datos (tablas/colecciones)
- `enumTypes` - Tipos enumerados
- `roles` - Roles de usuario y permisos
- `functions` - Procedimientos/funciones almacenadas
- `dataItems` - Archivos de datos
- `webPackages` - Paquetes de aplicaciones web
- `externalEntities` - Integraciones con sistemas externos
- `variables` - Variables globales
- `pythonPackages` - Dependencias de Python

### sistema de entidades

Las entidades son las estructuras de datos centrales con metadatos extensos.

#### atributos con configuración extensa

Tipos de datos soportados:
- `TEXT` (Texto)
- `BOOLEAN` (Booleano)
- `INTEGER` (Entero)
- `DECIMAL` (Decimal)
- `DATE` (Fecha)
- `TIMESTAMP` (Marca de tiempo)
- `SERIAL` (Auto-incremental)
- `TIME` (Hora)
- `POINT` (Geolocalización)
- `DOCUMENT` (Documento)
- `SIGNATURE` (Firma)
- `BARCODE` (Código de barras)
- `VECTOR` (Vector)

Propiedades de interfaz de usuario:
- `visible` (Visibilidad en formularios)
- `list` (Mostrar en listas)
- `order` (Orden de visualización)
- `group` (Agrupación lógica)
- `tab` (Pestaña de organización)
- `labels` (Etiquetas multiidioma)

Validación:
- `min`/`max` (Valores mínimo y máximo)
- `pattern` (Patrones de validación)
- `required` (Campos obligatorios)
- `length` (Longitud máxima)

Características especiales:
- `array` (Campos array)
- `computed` (Campos calculados)
- `encrypted` (Campos encriptados)
- `textSearch` (Búsqueda de texto)
- `barcodeType` (EAN_8, EAN_13, CODE_39, CODE_128, ITF, RSS_14, QR_CODE, DATA_MATRIX, PDF_417)

Internacionalización:
- Soporte de etiquetas en múltiples idiomas (más de 50 idiomas)

#### claves
- Claves primarias
- Restricciones únicas
- Índices de búsqueda de texto

#### referencias
- Relaciones de clave foránea con opciones de cascada
- `cascadeDelete` (Eliminación en cascada)
- `cascadeSetNull` (Establecer nulo en cascada)

#### grupos y pestañas
- Agrupación lógica de atributos y organización de UI

#### disparadores (triggers)
- Eventos: `INSERT`, `UPDATE`, `DELETE`
- Momentos: `BEFORE`, `AFTER`, `INSTEAD_OF`
- Alcance: `ROW`, `STATEMENT`

### capa de aplicación
- Las aplicaciones definen el sistema con codificación de colores, documentación y orden
- Etiquetas multiidioma para internacionalización
- Integración con entidades externas y paquetes web

### sistema de permisos

Roles definen niveles de acceso de usuario y se aplican permisos a diferentes niveles:
- Permisos de entidad: `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `MENU`
- Permisos de atributo: control a nivel de campo
- Políticas a nivel de fila: tipos `PERMISSIVE`/`RESTRICTIVE`, alcance `ALL`, `SELECT`, `INSERT`, `UPDATE`, `DELETE`
- Permisos de entidad externa

### sistema de funciones

Lenguajes soportados:
- `plpgsql` (PostgreSQL)
- `plpython3u` (Python en PostgreSQL)
- `ECMAScriptNashorn` (JavaScript)

Características:
- Endpoints HTTP, programación con cron
- Interceptación de solicitudes HTTP y de consultas `SELECT`
- Activables por eventos de entidad

### integraciones externas
- Entidades externas con autenticación y tokens de acceso
- Paquetes web (apps estáticas o Node.js)
- Elementos de datos (Excel y otras fuentes)
- Paquetes de Python

### características clave del DSL
- Internacionalización extensa (entidades, atributos, aplicaciones)
- Capacidades de UI/UX (grid responsivo, controles ricos, filtros, mapas, escaneo de códigos de barras)
- Validación de datos (tipos, rango, patrones, obligatorios, personalizada vía triggers)
- Seguridad (encriptación, permisos granulares, RLS, gestión de tokens externos)
- Lógica de negocio (triggers, campos calculados, indexación de texto, eventos)

### cómo las entidades referencian otras entidades

Definición básica dentro de una entidad usando `reference`:

```airflows
entity Demo.Product {
    // ... otros atributos ...
    
    reference category {
        attribute category
        basicFilter
        cascadeDelete
        label
        list
        listIsVisible
        referencedKey: Demo.Category.Category_pkey
        visible
    }
}
```

Componentes clave:
- `attribute`: atributo local que almacena la clave foránea
- `referencedKey`: clave específica en la entidad referenciada (`EntityName.KeyName`)

Opciones comunes:
- `basicFilter`, `cascadeDelete`, `cascadeSetNull`, `label`, `list`, `listIsVisible`, `listIsFilteredWhenEmpty`, `visible`, `linkDisabled`, `additionalAttributes`, `additionalFilter`, `group`, `tab`, `order`, `sm`, `xs`

Ejemplo completo:

```airflows
entity Demo.Product {
    attribute category {
        label es_ES: "Categoría"
        label en_US: "Category"
        type: INTEGER
    }
    
    reference category {
        attribute category
        basicFilter
        cascadeDelete
        label
        list
        listIsVisible
        referencedKey: Demo.Category.Category_pkey
        visible
    }
}
```

Funcionamiento práctico:
1. El atributo local almacena el ID de la entidad referenciada
2. La referencia habilita navegación y visualización en UI
3. Opciones de cascada mantienen integridad referencial
4. Las referencias permiten filtrado y búsqueda

### ejemplo (demo)
- Entidad `Category`: categorías con imágenes y búsqueda
- Entidad `Product`: productos con precios, códigos de barras, género
- Enum `Gender`: masculino/femenino con colores e iconos
- Rol `Customer`: permisos específicos
- Funciones: actualización de búsqueda de texto y cálculos de ranking
- Enlaces externos: integración con sistemas externos

### estructura creada en base de datos al definir referencias desde la UI

Cuando el usuario crea una referencia desde la UI (asociando atributos mediante el diagrama E/R), en la base de datos se crea:

```
EntityAttribute (creado por el usuario previamente)
id|container|name   |type   |isArray|
--+---------+-------+-------+-------+
65|        3|id     |SERIAL |false  |
68|        3|id_user|INTEGER|false  | 

EntityReference (creado en el momento de establecer la relación)
id|container|name   |referencedKey|isList|
--+---------+-------+-------------+------+
 1|        3|id_user|            4|true  |
```

### estados y ciclo de vida al crear/editar/eliminar referencias

Cuando el usuario opera una referencia desde la UI, se observan estos estados en las tablas base:

- Creación inicial:
  - `EntityAttribute`: el atributo FK ya existe (creado previamente por el usuario o por el asistente de atributos).
  - `EntityReference`: se inserta un nuevo registro con `container` (entidad origen), `name` (nombre de la referencia), `referencedKey` (clave en la entidad destino) y flags como `isList`.

- Edición de referencia:
  - `EntityReference`: UPDATE de campos configurables (por ejemplo `referencedKey`, `isList`, opciones de cascada, configuración de UI como `listIsVisible`, `basicFilter`, etc.).
  - `EntityAttribute`: sin cambios estructurales, salvo que el usuario cambie el atributo FK; en ese caso puede haber UPDATE de `name` o metadatos.

- Eliminación de referencia:
  - `EntityReference`: DELETE del registro correspondiente.
  - `EntityAttribute`: se mantiene, ya que el atributo FK sigue existiendo a menos que el usuario lo elimine explícitamente.

Notas:
- La integridad referencial (FK y cascadas) se materializa a nivel de esquema con constraints y triggers según la configuración del DSL.
- La UI puede reflejar propiedades (`linkDisabled`, `additionalFilter`, `additionalAttributes`) que no cambian la estructura física pero sí la representación y navegación.

## Fuentes de datos

### 1. Modelo persistido (base de datos → GraphQL)

**Origen**: Esquema y estructura de la base de datos  
**Consultas GraphQL**: 
- `GetModel` - Retorna el esquema de la base de datos como string JSON
- `GetAllData` - Retorna configuraciones de entidades, aplicaciones, entidades externas, etc.

**Flujo de datos**:
```
Base de datos → API GraphQL → getModel() → DBParsedModel
Base de datos → API GraphQL → getModelData() → GetAllDataQuery
```

**Características clave**:
- Contiene el esquema real de la base de datos (tablas, columnas, relaciones, restricciones)
- Incluye definiciones de entidades, atributos, claves, referencias
- Proporciona la base estructural del modelo de datos
- Se almacena en `rawDBModel` y `rawAllModelData` en el estado de la aplicación

**Archivos relevantes**:
- `src/data/queries/getModel.ts`
- `src/data/queries/getModelData.ts`
- `src/data/queries/queriesGraphql/getModel.graphql`
- `src/data/queries/queriesGraphql/getAllData.graphql`

### 2. Modelo local (frontend hardcodeado)

**Origen**: `src/data/mocks/localModelJson.json` (archivo JSON hardcodeado)  
**Procesamiento**: Transformado por `transformLocalModel()` en `localModel.ts`

**Características clave**:
- Contiene configuraciones específicas del frontend no almacenadas en la base de datos
- Incluye preferencias de UI, configuraciones de visualización y metadatos solo del frontend
- Actúa como respaldo cuando los datos del backend no están disponibles
- Se almacena en `localModel` en el estado de la aplicación

**Archivos relevantes**:
- `src/data/mocks/localModelJson.json`
- `src/data/mocks/localModel.ts`
- `src/data/mocks/type.ts`

### 3. Mejoras en tiempo de ejecución (datos combinados)

**Origen**: Combinación de modelo persistido + modelo local + configuraciones del backend  
**Procesamiento**: Función `parseModels()` combina todas las fuentes de datos

**Características clave**:
- Combina el esquema de la base de datos con las configuraciones del frontend
- Añade propiedades de referencia como `linkDisabled`, `additionalFilter`, `additionalAttributes`
- Procesa entidades externas y aplicaciones
- Crea el modelo de datos final funcional

**Archivos relevantes**:
- `src/components/bootstrap/all-models/parseModels.ts`

## Arquitectura de selectores

La aplicación utiliza tres tipos de selectores para acceder a estos datos:

### 1. Selectores DB (`selectorsDB.ts`)

**Propósito**: Acceder a la estructura del modelo de base de datos en bruto  
**Fuente de datos**: `DBParsedModel` (de la consulta GraphQL `GetModel`)  
**Funciones clave**:
- `getInstance()` - Obtener definición de entidad del esquema de la base de datos
- `getInstanceAttributes()` - Obtener atributos de la base de datos
- `isReferenceField()` - Verificar si un campo es una referencia de base de datos
- `getReferenceEntityAndSchema()` - Obtener información de la entidad referenciada
- `getInstanceAttributePrimaryKey()` - Obtener la clave primaria de una entidad
- `getReferenceEntityName()` - Obtener el nombre de la entidad referenciada

**Casos de uso**:
- Validación de estructura de base de datos
- Verificación de relaciones entre entidades
- Análisis de esquemas de base de datos

### 2. Selectores Model (`selectorsModels.ts`)

**Propósito**: Acceder a datos del modelo combinado (base de datos + configuración del frontend)  
**Fuentes de datos**: `GetAllDataQuery` + `DBParsedModel` (procesado)  
**Funciones clave**:
- `getInstance()` - Obtener entidad con respaldo al modelo local
- `getInstanceAttributesDictionary()` - Obtener atributos con mejoras en tiempo de ejecución
- `getApp()` - Obtener configuración de aplicación
- `getExternalEntity()` - Obtener información de entidad externa
- `isExternalEntity()` - Verificar si una entidad es externa
- `getQuestions()` - Obtener preguntas asociadas a una entidad
- `getInstanceAttributesWithEventStart/End()` - Obtener atributos de eventos

**Características especiales**:
- Implementa lógica de respaldo: si no encuentra datos en el backend, usa el modelo local
- Combina atributos base con propiedades añadidas en tiempo de ejecución
- Maneja entidades externas y aplicaciones

### 3. Selectores Global (`selectorsGlobal.ts`)

**Propósito**: Acceso unificado a todas las fuentes de datos  
**Fuentes de datos**: Las tres fuentes de datos combinadas  
**Estructura**:
```typescript
{
  model: createSelectorsModels(rawAllModelData, rawDBModel),
  db: createSelectorsDB(rawDBModel),
  getErrorMessagesList: () => errorMessagesList
}
```

**Funciones clave**:
- Proporciona acceso unificado a todos los selectores
- Centraliza la gestión de mensajes de error
- Actúa como punto de entrada principal para el acceso a datos

## Flujo de datos

### Diagrama principal del flujo de datos

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Base de datos │    │  Modelo Local   │    │  APIs Backend   │
│   (Esquema)     │    │  (Archivo JSON) │    │  (Configuración)│
└─────────┬───────┘    └─────────┬───────┘    └─────────┬───────┘
          │                      │                      │
          │ GetModel             │                      │ GetAllData
          │                      │                      │
          ▼                      ▼                      ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   DBParsedModel │    │   LocalModel    │    │ GetAllDataQuery │
│   (rawDBModel)  │    │                 │    │(rawAllModelData)│
└─────────┬───────┘    └─────────┬───────┘    └─────────┬───────┘
          │                      │                      │
          │                      │                      │
          └──────────────────────┼──────────────────────┘
                                 │
                                 ▼
                    ┌─────────────────┐
                    │   parseModels() │
                    │   (Combinador)  │
                    └─────────┬───────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ Modelo Mejorado │
                    │ (Estado Final)  │
                    └─────────┬───────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │   Selectores    │
                    │ (Capa de Acceso)│
                    └─────────────────┘
```

### Diagrama de arquitectura de selectores

```
┌─────────────────────────────────────────────────────────────────┐
│                    Selectores Globales                          │
│                    (selectorsGlobal.ts)                        │
└─────────────────────┬───────────────────────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
        ▼             ▼             ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│   Model     │ │     DB      │ │   Errors    │
│ Selectors   │ │ Selectors   │ │   List      │
│             │ │             │ │             │
└─────┬───────┘ └─────┬───────┘ └─────────────┘
      │               │
      │               │
      ▼               ▼
┌─────────────┐ ┌─────────────┐
│   Fuentes   │ │   Fuentes   │
│   de Datos  │ │   de Datos  │
│             │ │             │
│ • GetAllData│ │ • DBParsed  │
│   Query     │ │   Model     │
│ • DBParsed  │ │             │
│   Model     │ │             │
│ • Local     │ │             │
│   Model     │ │             │
│   (respaldo)│ │             │
└─────────────┘ └─────────────┘
```

## Proceso de mejora de datos

### Función `parseModels()`

Esta es la función clave que combina todas las fuentes de datos:

**Entrada**:
- `model`: DBParsedModel (esquema de base de datos)
- `allModelData`: GetAllDataQuery (configuraciones del backend)
- `contextState`: AppState (estado de la aplicación)
- `logout`: función de cierre de sesión

**Procesos principales**:

1. **Procesamiento de entidades externas**:
   ```typescript
   if (allModelData.externalEntities) {
     localModel.entities = {
       ...localModel.entities,
       ...processExternalEntities({...})
     }
   }
   ```

2. **Procesamiento de aplicaciones**:
   ```typescript
   if (allModelData.applications) {
     allModelData.applications.forEach((application) => {
       localModel.applications[application.name] = {
         order: application.order,
         color: application.color,
       }
     })
   }
   ```

3. **Procesamiento de entidades**:
   - Combina datos de base de datos con configuraciones del frontend
   - Añade propiedades de referencia (`linkDisabled`, `additionalFilter`, etc.)
   - Procesa referencias directas e inversas

4. **Mejora de atributos de referencia**:
   ```typescript
   // Añade linkDisabled y otras propiedades a los atributos del modelo backend
   (attribute as any).linkDisabled = !!directRef.linkDisabled
   (attribute as any).additionalFilter = directRef.additionalFilter
   (attribute as any).additionalAttributes = directRef.additionalAttributes
   ```

## Mecanismo de respaldo

### En selectores de modelo

Los selectores de modelo implementan un mecanismo de respaldo robusto:

```typescript
const getInstance = (...args: EntityParams): ModelEntity => {
  const [schema, name] = parseEntity(...args)
  const instance = data?.entities?.find((e) => e?.schema === schema && e?.name === name)
  return (instance ?? localModelRaw[`${schema}.${name}`]) as ModelEntity
}
```

**Lógica de respaldo**:
1. Intenta encontrar la entidad en los datos del backend (`GetAllDataQuery`)
2. Si no la encuentra, usa el modelo local como respaldo
3. Esto asegura que la aplicación funcione incluso con conectividad limitada del backend

## Tipos de datos principales

### DBParsedModel
```typescript
interface DBParsedModel {
  name: string
  schemaNames: Record<string, string>
  entities: Record<string, DBEntityDefinition>
  externalEntities: DBEntityDefinition[]
  customTypes: Record<string, DBCustomTypeDefinition>
  enumTypes: Record<string, DBEnumTypeDefinition>
  documentation: string
  edition: string
  super: boolean
  defaultEntity: string | null
  defaultExternalEntity: string | null
  disableReferenceWidgetButtons: boolean
  agreementAccepted: boolean
  username: string
  email: string | null
  dashboards: DBDashboardDefinition[]
}
```

### GetAllDataQuery
Contiene:
- `entities`: Lista de entidades con sus configuraciones
- `applications`: Lista de aplicaciones
- `externalEntities`: Entidades externas
- `dashboards`: Configuraciones de dashboards
- `enumTypes`: Tipos de enumeración
- `errorMessages`: Mensajes de error
- `variables`: Variables del sistema

### LocalModel
```typescript
type LocalModel = {
  edition: string
  entities: Record<string, EntityConfig>
  applications?: Record<string, ApplicationConfig>
}
```

## Casos de uso y ejemplos

### 1. Obtener una entidad con respaldo
```typescript
const selectors = createSelectorsModels(backendData, processedModel)
const entity = selectors.getInstance('Schema.EntityName')
// Si no existe en backend, usa localModel como respaldo
```

### 2. Verificar si un campo es una referencia
```typescript
const dbSelectors = createSelectorsDB(dbModel)
const result = dbSelectors.isReferenceField('Schema.Entity', 'fieldName')
// Retorna: { isReference: boolean, referencedEntity?: string }
```

### 3. Obtener atributos con mejoras en tiempo de ejecución
```typescript
const modelSelectors = createSelectorsModels(backendData, processedModel)
const attributes = modelSelectors.getInstanceAttributesDictionary('Schema.Entity')
// Incluye propiedades como linkDisabled, additionalFilter, etc.
```

## Consideraciones técnicas

### Rendimiento
- Los selectores se crean una vez y se reutilizan
- Los datos se procesan en `parseModels()` para evitar procesamiento repetido
- El mecanismo de respaldo es eficiente (búsqueda directa en objetos)

### Mantenibilidad
- Separación clara de responsabilidades entre selectores
- Tipos TypeScript bien definidos
- Funciones puras que facilitan las pruebas

### Robustez
- Mecanismo de respaldo para casos de conectividad limitada
- Validación de datos en múltiples niveles
- Manejo de errores en `parseModels()`

## Archivos clave del proyecto

### Selectores
- `src/contexts/selectors/selectorsDB.ts` - Selectores de base de datos
- `src/contexts/selectors/selectorsModels.ts` - Selectores de modelo
- `src/contexts/selectors/selectorsGlobal.ts` - Selectores globales

### Procesamiento de datos
- `src/components/bootstrap/all-models/parseModels.ts` - Función principal de combinación
- `src/components/bootstrap/all-models/useFetchModels.ts` - Hook para obtener datos

### Fuentes de datos
- `src/data/queries/getModel.ts` - Obtener modelo de base de datos
- `src/data/queries/getModelData.ts` - Obtener datos de configuración
- `src/data/mocks/localModel.ts` - Modelo local
- `src/data/mocks/localModelJson.json` - Datos hardcodeados

### Tipos
- `src/data/model/GetModelType.ts` - Tipos del modelo de base de datos
- `src/data/mocks/type.ts` - Tipos del modelo local
- `src/generated/graphql.ts` - Tipos generados de GraphQL

## Conclusión

La arquitectura del modelo de datos de Airflows frontend está bien diseñada con una separación clara de responsabilidades. La combinación de modelo persistido, modelo local y mejoras en tiempo de ejecución proporciona flexibilidad y robustez, mientras que el sistema de selectores ofrece una interfaz unificada y eficiente para acceder a los datos.

Esta arquitectura permite:
- **Flexibilidad**: Configuraciones específicas del frontend separadas de la estructura de la base de datos
- **Robustez**: Mecanismo de respaldo para casos de conectividad limitada
- **Mantenibilidad**: Código bien estructurado con tipos TypeScript claros
- **Rendimiento**: Procesamiento eficiente y reutilización de selectores
