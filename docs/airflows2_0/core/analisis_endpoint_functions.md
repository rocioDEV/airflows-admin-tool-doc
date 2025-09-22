# Análisis del endpoint `/functions/` - Ejecución de funciones almacenadas en base de datos

## Introducción

El endpoint `/functions/` de Airflows permite la ejecución programática de funciones almacenadas en base de datos. Este endpoint soporta tres tipos de lenguajes de programación diferentes y maneja la ejecución, el procesamiento de parámetros y la gestión de resultados de manera unificada.

## Arquitectura del endpoint

### Configuración de rutas

El endpoint está configurado en `com.niledb.platform/src/main/java/verticles/HttpVerticle.java` con las siguientes rutas:

```java
router.get("/functions/:functionName").blockingHandler(FunctionsHandler::execute);
router.post("/functions/:functionName").handler(BodyHandler.create()).blockingHandler(FunctionsHandler::execute);
router.options("/functions/:functionName").blockingHandler(routingContext -> { /* CORS */ });
```

### Componentes principales

1. **FunctionsHandler**: Maneja las peticiones HTTP, evalúa scripts JavaScript y ejecuta funciones de base de datos
2. **FunctionsHelper**: Gestiona la carga de funciones y flags (HTTP, interceptores)
3. **DatabaseHelper**: Gestiona conexiones a base de datos para funciones PL

## Lenguajes soportados

### 1. ECMAScriptNashorn (JavaScript)

**Características:**
- Motor JavaScript de la JVM (ScriptEngine name: "JavaScript")
- Acceso completo al contexto HTTP (request/response)
- Ejecución directa en el servidor
- No hay captura automática de valores de retorno; el script debe escribir la respuesta

**Implementación:**
```java
// helpers/FunctionsHelper.java
public static ScriptEngine engine = new ScriptEngineManager().getEngineByName("JavaScript");

// helpers/FunctionsHandler.java
ScriptContext context = new SimpleScriptContext();
context.setAttribute("routingContext", routingContext, ScriptContext.ENGINE_SCOPE);
FunctionsHelper.engine.eval(FunctionsHelper.functions.get(functionName), context);
```

#### Ejecución JavaScript

Actualmente las funciones JavaScript deben escribir y cerrar explícitamente la respuesta HTTP usando `routingContext.response()`. No hay captura automática de valores de retorno ni detección de modos.

```javascript
function myFunction() {
    var request = routingContext.request();
    var name = request.getParam("name") || "world";

    routingContext.response()
        .putHeader("Content-Type", "application/json")
        .end(JSON.stringify({ message: "Hello, " + name + "!" }));
}
```

Para POST, el body JSON está disponible porque la ruta usa `BodyHandler`:

```javascript
function create() {
    var body = routingContext.body().asJsonObject();
    var value = body.getString("value");

    routingContext.response()
        .putHeader("Content-Type", "application/json")
        .end(JSON.stringify({ created: value }));
}
```

Errores en el script deben manejarse dentro del propio script (por ejemplo, estableciendo `statusCode` y enviando un cuerpo de error). Si ocurre una excepción en la evaluación del script, el servidor finaliza la respuesta sin un JSON de error estructurado.


#### Guía de testing

##### Test 1: Función que escribe a la respuesta
```javascript
function myLegacyFunction() {
    var request = routingContext.request();
    var param = request.getParam("test");
    
    // Escribir directamente a la respuesta
    routingContext.response()
        .putHeader("Content-Type", "application/json")
        .end(JSON.stringify({message: "Hello, " + param + "!"}));
}
```

**Respuesta esperada:**
```json
{"message": "Hello, world!"}
```


### 2. PL/pgSQL

**Características:**
- Lenguaje procedural nativo de PostgreSQL
- Ejecución directa en la base de datos
- Soporte para parámetros tipados

**Implementación:**
```java
// helpers/FunctionsHandler.java
Connection connection = DatabaseHelper.getConnection(accessToken);
PreparedStatement ps = connection.prepareStatement(
    "SELECT \"" + functionName.split("\\.")[0] + "\".\"" + functionName.split("\\.")[1] +
    "\"(" + (parameterNames != null ? String.join(", ", "?".repeat(parameterNames.length).split("")) : "") + ") as \"result\"");
// Vinculación por tipos (text, int, numeric, boolean, date) y valores null
ResultSet rs = ps.executeQuery();
ResultSetMetaData rsmd = rs.getMetaData();
int columnCount = rsmd.getColumnCount();
JsonArray jsonArray = new JsonArray();
while (rs.next()) {
    JsonObject jsonObject = new JsonObject();
    for (int i = 1; i <= columnCount; i++) {
        String columnName = rsmd.getColumnLabel(i);
        Object value = rs.getObject(i);
        jsonObject.put(columnName, value);
    }
    jsonArray.add(jsonObject);
}
response.end(jsonArray.encodePrettily());
```

### 3. PL/Python3U

**Características:**
- Extensión Python para PostgreSQL
- Acceso a librerías Python desde la base de datos
- Ejecución híbrida (Python + SQL)
- **Importante**: Siempre retorna resultados en formato array

**Implementación:**
- Misma implementación que PL/pgSQL
- Se ejecuta como función de base de datos
- Soporte para tipos de datos Python

#### Ejecución de funciones Python

Las funciones Python se ejecutan como funciones de base de datos usando una declaración `SELECT`:

```java
// helpers/FunctionsHandler.java
PreparedStatement ps = connection.prepareStatement(
    "SELECT \"" + functionName.split("\\.")[0] + "\".\"" + functionName.split("\\.")[1] + "\"(" + 
    (parameterNames != null ? String.join(", ", "?".repeat(parameterNames.length).split("")) : "") + 
    ") as \"result\"");
```

#### Procesamiento de resultados Python

El sistema procesa el `ResultSet` y convierte cada fila a un objeto JSON:

```java
// helpers/FunctionsHandler.java
while (rs.next()) {
    JsonObject jsonObject = new JsonObject();
    for (int i = 1; i <= columnCount; i++) {
        String columnName = rsmd.getColumnLabel(i);
        Object value = rs.getObject(i);
        jsonObject.put(columnName, value);
    }
    jsonArray.add(jsonObject);
}
```

#### Formato de respuesta estándar para Python

**Las funciones Python SIEMPRE retornan sus resultados en formato array:**

```json
[
  {
    "result": "valor_devuelto_por_la_funcion_python"
  }
]
```

#### ¿Por qué Python siempre retorna arrays?

1. **Ejecución como función de base de datos**: Las funciones Python se tratan como funciones de base de datos (PL/Python3U), no como scripts Python independientes
2. **Procesamiento de ResultSet**: El sistema procesa el resultado de la base de datos, que naturalmente crea una estructura de array
3. **API consistente**: Esto proporciona un formato de respuesta consistente entre todas las funciones de base de datos (PL/pgSQL y PL/Python3U)

#### Comparación con funciones JavaScript

Las funciones JavaScript escriben directamente a la respuesta HTTP con el formato que el script decida. Las funciones Python siempre se procesan a través de la capa de base de datos, lo que impone el formato de array.

**Conclusión**: Las funciones Python en el endpoint `/functions/` siempre retornarán sus resultados envueltos en una estructura de array JSON.

## Procesamiento de parámetros

### Para funciones JavaScript (ECMAScriptNashorn)

Los parámetros se pasan a través del contexto HTTP con tres métodos diferentes:

#### 1. Query parameters
```javascript
function myFunction() {
    var request = routingContext.request();
    var param = request.getParam("parameterName");
    routingContext.response().putHeader("Content-Type", "application/json").end(JSON.stringify({result: "Hello, " + param + "!"}));
}
```

**Uso:** `GET /functions/myFunction?parameterName=value`

#### 2. HTTP headers
```javascript
function myFunction() {
    var request = routingContext.request();
    var param = request.getHeader("parameterName");
    routingContext.response().putHeader("Content-Type", "application/json").end(JSON.stringify({result: "Hello, " + param + "!"}));
}
```

**Uso:** Incluir header `parameterName: value` en la petición HTTP

#### 3. JSON body (para peticiones POST)
```javascript
function myFunction() {
    var body = routingContext.body().asJsonObject();
    var param = body.getString("parameterName");
    routingContext.response().putHeader("Content-Type", "application/json").end(JSON.stringify({result: "Hello, " + param + "!"}));
}
```

**Uso:** POST a `/functions/myFunction` con JSON body:
```json
{
    "parameterName": "value"
}
```

#### Ejemplo completo: Función JavaScript con múltiples fuentes de parámetros
```javascript
function processData() {
    var request = routingContext.request();
    var body = routingContext.body().asJsonObject();
    
    var queryParam = request.getParam("queryParam");
    var headerParam = request.getHeader("headerParam");
    var bodyParam = body.getString("bodyParam");
    
    var result = { query: queryParam, header: headerParam, body: bodyParam };
    routingContext.response().putHeader("Content-Type", "application/json").end(JSON.stringify({ result: result }));
}
```

**Uso completo:**
```bash
# Petición con todos los tipos de parámetros
curl -X POST "http://localhost:8080/functions/processData?queryParam=fromQuery" \
  -H "headerParam: fromHeader" \
  -H "Content-Type: application/json" \
  -d '{"bodyParam": "fromBody"}'
```

**Respuesta esperada:**
```json
[
  {
    "result": {
      "query": "fromQuery",
      "header": "fromHeader", 
      "body": "fromBody"
    }
  }
]
```

### Para funciones de base de datos (PL/pgSQL y PL/Python3U)

Los parámetros se procesan de manera estructurada con tipos predefinidos:

#### Definición de parámetros
Los parámetros se definen en la tabla `Models.Function` con el campo `customParameters`:
```sql
-- Ejemplo: "param1 text, param2 int, param3 boolean"
```

#### Tipos de parámetros soportados
- `text` - Valores de texto
- `int` - Valores enteros  
- `numeric` - Números decimales
- `boolean` - Valores verdadero/falso
- `date` - Valores de fecha

#### Paso de parámetros
Los parámetros se pasan en el orden en que están definidos:

**Petición GET:**
```
GET /functions/schema.functionName?param1=value1&param2=123&param3=true
```

**Petición POST:**
```json
{
    "param1": "value1",
    "param2": 123,
    "param3": true
}
```

#### Ejemplo completo: Función PL/pgSQL con parámetros tipados
```sql
CREATE OR REPLACE FUNCTION "MySchema"."processData"(
    param1 text,
    param2 int,
    param3 boolean
)
RETURNS text AS $$
BEGIN
    RETURN 'Processed: ' || param1 || ', ' || param2 || ', ' || param3;
END;
$$ LANGUAGE plpgsql;
```

**Uso:**
```bash
# Petición GET
curl "http://localhost:8080/functions/MySchema.processData?param1=hello&param2=42&param3=true"

# Petición POST
curl -X POST "http://localhost:8080/functions/MySchema.processData" \
  -H "Content-Type: application/json" \
  -d '{"param1": "hello", "param2": 42, "param3": true}'
```

**Respuesta esperada:**
```json
[
  {
    "result": "Processed: hello, 42, true"
  }
]
```

#### Ejemplo completo: Función PL/Python3U con parámetros
```python
CREATE OR REPLACE FUNCTION "MySchema"."analyzeData"(
    input_text text,
    threshold int
)
RETURNS text AS $$
    import json
    
    # Procesar los parámetros
    result = {
        "input": input_text,
        "threshold": threshold,
        "analysis": "Data processed successfully"
    }
    
    return json.dumps(result)
$$ LANGUAGE plpython3u;
```

**Uso:**
```bash
curl "http://localhost:8080/functions/MySchema.analyzeData?input_text=sample&threshold=10"
```

**Respuesta esperada:**
```json
[
  {
    "result": "{\"input\": \"sample\", \"threshold\": 10, \"analysis\": \"Data processed successfully\"}"
  }
]
```

#### Cómo se parsean los parámetros (PL/pgSQL y PL/Python3U)

- Definición: se activa sólo si la función está marcada como personalizada (`isCustomFunction=true`) y define `customParameters` en `Models.Function`.
- Formato de `customParameters`: lista separada por comas con nombre y tipo por parámetro. Ej.: `"param1 text, param2 int, access_token text"`. Se eliminan comillas y espacios extra.
- Precedencia de fuentes por nombre de parámetro: JSON body > query string > header.
  - El body JSON sólo se considera si `Content-Type` es exactamente `application/json`.
- Parámetro especial `access_token`: si existe en `customParameters`, se rellena automáticamente con el token extraído de `Authorization` (se toma la segunda parte al dividir por espacio, p. ej. `Bearer <token>`).
- Tipos soportados: `text`, `int`, `numeric`, `boolean`, `date`. Cualquier otro tipo se vincula como `VARCHAR`.
- Conversión de tipos: si viene del body JSON, se respeta el tipo nativo (boolean, number, string). Si viene de query/header (string), el driver realiza la conversión al tipo indicado al hacer `setObject(..., Types.XXX)`.
- Nulos: si un parámetro no llega en ninguna fuente, se envía `NULL` con el tipo indicado (`setNull(..., Types.XXX)`).
- Orden en la SQL: los placeholders `?` se generan en el mismo orden que aparecen en `customParameters` (aunque los valores se busquen por nombre).
- Si no hay `customParameters`, la invocación se hace sin placeholders (no se admite paso de parámetros por nombre en ese caso).

Código relevante (simplificado):

```java
// helpers/FunctionsHandler.java - Resolución de fuentes y tipos
String[] parameters = FunctionsHelper.plCustomFunctionsParameters.get(functionName).split(",");
for (int i = 0; i < parameters.length; i++) {
    parameters[i] = parameters[i].trim().replaceAll("\"", "").trim();
    parameterNames[i] = parameters[i].split("\\s+")[0];
    parameterTypes[i] = parameters[i].split("\\s+")[1];

    Object jsonValue = jsonBody != null ? jsonBody.getValue(parameterNames[i]) : null;
    String headerValue = request.getHeader(parameterNames[i]);
    String parameterValue = request.getParam(parameterNames[i]);

    parameterValues[i] = jsonValue != null ? jsonValue : (parameterValue != null ? parameterValue : headerValue);
    if (parameterNames[i].equals("access_token")) {
        parameterValues[i] = accessToken; // extraído de Authorization
    }
}

// helpers/FunctionsHandler.java - Vinculación por tipo y manejo de nulls
if (parameterValues[i] != null) {
    switch (parameterTypes[i]) {
        case "text": ps.setObject(i + 1, parameterValues[i], Types.VARCHAR); break;
        case "int": ps.setObject(i + 1, parameterValues[i], Types.INTEGER); break;
        case "numeric": ps.setObject(i + 1, parameterValues[i], Types.NUMERIC); break;
        case "boolean": ps.setObject(i + 1, parameterValues[i], Types.BOOLEAN); break;
        case "date": ps.setObject(i + 1, parameterValues[i], Types.DATE); break;
        default: ps.setObject(i + 1, parameterValues[i], Types.VARCHAR);
    }
} else {
    switch (parameterTypes[i]) {
        case "text": ps.setNull(i + 1, Types.VARCHAR); break;
        case "int": ps.setNull(i + 1, Types.INTEGER); break;
        case "numeric": ps.setNull(i + 1, Types.NUMERIC); break;
        case "boolean": ps.setNull(i + 1, Types.BOOLEAN); break;
        case "date": ps.setNull(i + 1, Types.DATE); break;
        default: ps.setNull(i + 1, Types.VARCHAR);
    }
}
```

#### Mapeo de tipos (PL)

| Tipo `customParameters` | JDBC Type usado | Valor esperado si viene de body JSON | Valor esperado si viene de query/header |
|-------------------------|-----------------|--------------------------------------|----------------------------------------|
| `text`                  | `Types.VARCHAR` | string                               | string                                 |
| `int`                   | `Types.INTEGER` | number (entero)                       | string convertible a entero            |
| `numeric`               | `Types.NUMERIC` | number                                | string convertible a número            |
| `boolean`               | `Types.BOOLEAN` | boolean                               | `"true"`/`"false"`                    |
| `date`                  | `Types.DATE`    | string fecha compatible JDBC          | string fecha compatible JDBC           |

Notas:
- Si el valor no llega, se envía `NULL` del tipo correspondiente.
- Tipos no listados se envían como `Types.VARCHAR`.

#### Ejemplos de combinación de fuentes

```bash
# helpers/FunctionsHandler.java - Ejemplo de invocación
# Suponga customParameters: "q text, flag boolean, threshold int, access_token text"

# 1) Prioridad JSON sobre query/header
curl -X POST "http://localhost:8080/functions/MySchema.myFunc?q=fromQuery&flag=true" \
  -H "flag: false" \
  -H "Authorization: Bearer XYZ" \
  -H "Content-Type: application/json" \
  -d '{"q":"fromBody","threshold": 10}'

# Resultado de binding:
# q -> "fromBody" (JSON)
# flag -> "true" (query) porque JSON no lo define; se convierte a boolean
# threshold -> 10 (JSON)
# access_token -> "XYZ" (Authorization)

# 2) Sin JSON, cae a query y luego header
curl "http://localhost:8080/functions/MySchema.myFunc?q=fromQuery&threshold=7" \
  -H "flag: false" \
  -H "Authorization: Bearer XYZ"

# Resultado de binding:
# q -> "fromQuery"
# flag -> "false" (header)
# threshold -> 7 (query)
# access_token -> "XYZ"
```

### Comparación de métodos de paso de parámetros

| Aspecto | JavaScript | PL/pgSQL y PL/Python3U |
|---------|------------|------------------------|
| **Flexibilidad** | Alta - 3 métodos (query, headers, body) | Baja - solo parámetros ordenados |
| **Tipos de datos** | Dinámicos (string, number, boolean, object) | Predefinidos (text, int, numeric, boolean, date) |
| **Validación** | Manual en la función | Automática por tipo |
| **Orden** | No importa | Importante (orden de definición) |
| **Métodos HTTP** | GET y POST | GET y POST |
| **Ejemplo GET** | `?param1=value1&param2=value2` | `?param1=value1&param2=value2` |
| **Ejemplo POST** | JSON body flexible | JSON body con parámetros ordenados |

### Guía de selección de método

#### Usar JavaScript cuando:
- Necesitas flexibilidad en el formato de parámetros
- Quieres combinar query parameters, headers y body
- Los parámetros son opcionales o variables
- Necesitas procesamiento complejo de entrada

#### Usar PL/pgSQL/PL/Python3U cuando:
- Los parámetros son fijos y bien definidos
- Necesitas validación automática de tipos
- Quieres consistencia en la API
- Los parámetros son obligatorios y en orden específico

## Gestión de resultados

### Para funciones JavaScript

**Características:**
- **Nuevo**: Soporte para captura de valores de retorno con formato JSON consistente
- **Legacy**: Las funciones pueden escribir directamente a la respuesta HTTP
- **Nuevo**: Detección automática del modo de ejecución
- Control total sobre el formato de respuesta (en modo direct response)

#### Modo Return Value (Nuevo)
```java
// En com.niledb.platform/src/main/java/helpers/FunctionsHelper.java
// El sistema captura el valor de retorno y lo formatea como JSON
if (result != null && !result.toString().isEmpty()) {
    return new JavaScriptFunctionResult(result.toString(), ExecutionMode.RETURN_VALUE);
}
```

**Formato de respuesta:**
```json
[
  {
    "result": "valor_retornado_por_la_funcion"
  }
]
```

#### Modo Direct Response (Legacy)
```java
// En com.niledb.platform/src/main/java/workflows/usecase/FunctionExecutor.java (líneas 79-80)
// JS script writes directly to HTTP response; nothing to return
return Future.succeededFuture("");
```

**Características:**
- Las funciones JavaScript escriben directamente a la respuesta HTTP
- No hay procesamiento adicional de resultados
- Control total sobre el formato de respuesta

### Para funciones de base de datos (PL/pgSQL y PL/Python3U)

**Procesamiento de resultados (PL):**

1. **Captura del ResultSet:**
```java
// En com.niledb.platform/src/main/java/helpers/FunctionsHandler.java (líneas 178-180)
ResultSet rs = ps.executeQuery();
ResultSetMetaData rsmd = rs.getMetaData();
int columnCount = rsmd.getColumnCount();
```

2. **Conversión a JSON:**
```java
// En com.niledb.platform/src/main/java/helpers/FunctionsHandler.java (líneas 181-193)
JsonArray jsonArray = new JsonArray();
while (rs.next()) {
    JsonObject jsonObject = new JsonObject();
    for (int i = 1; i <= columnCount; i++) {
        String columnName = rsmd.getColumnLabel(i);
        Object value = rs.getObject(i);
        jsonObject.put(columnName, value);
    }
    jsonArray.add(jsonObject);
}
```

3. **Formato de respuesta:**
```java
// helpers/FunctionsHandler.java
response.end(jsonArray.encodePrettily());
```

### Formato de respuesta estándar (PL)

Para funciones de base de datos (PL/pgSQL y PL/Python3U), el resultado se formatea como:

```json
[
  {
    "result": "valor_devuelto_por_la_funcion"
  }
]
```

#### Diferencias clave entre JavaScript y funciones de base de datos

| Aspecto | JavaScript | PL/pgSQL y PL/Python3U |
|---------|------------|------------------------|
| **Flexibilidad de formato** | Alta - script decide el formato | Baja - siempre formato array |
| **Modo return value** | ❌ No disponible | ✅ Siempre aplicado |
| **Modo direct response** | ✅ Sí | ❌ No |
| **Control de headers** | ✅ Completo (a cargo del script) | ❌ Limitado |
| **Formato de respuesta** | Variable según modo | Siempre `[{"result": "valor"}]` |
| **Ejecución** | En memoria (JVM) | En base de datos |
| **Overhead** | Mínimo | Mayor (conexión DB) |

#### Implicaciones prácticas

1. **JavaScript**: Más flexible, puede adaptarse a diferentes necesidades de respuesta
2. **PL/pgSQL y PL/Python3U**: Más consistente, siempre el mismo formato, pero menos flexible
3. **Python específicamente**: Siempre retorna arrays, sin excepciones, debido a su procesamiento como función de base de datos

## Configuración de funciones

### Carga de funciones desde base de datos

Las funciones se cargan dinámicamente desde la tabla `Models.Function`:

```sql
-- Para JavaScript
SELECT schema, name, contents, "cronExpression", "isHttpEnabled", 
       "interceptHttpRequests", "interceptSelects" 
FROM "Models"."Function" 
WHERE language = 'ECMAScriptNashorn'

-- Para PL/pgSQL y PL/Python3U
SELECT schema, name, contents, "cronExpression", "isHttpEnabled", 
       "interceptHttpRequests", "interceptSelects", "isCustomFunction", 
       "customParameters", "customReturnType" 
FROM "Models"."Function" 
WHERE language = 'plpython3u' OR language = 'plpgsql'
```

### Configuraciones disponibles

- **isHttpEnabled**: Habilita el endpoint HTTP
- **interceptHttpRequests**: Intercepta peticiones HTTP
- **interceptSelects**: Intercepta consultas SELECT
- **isCustomFunction**: Indica si es función personalizada
- **customParameters**: Definición de parámetros personalizados
- **customReturnType**: Tipo de retorno personalizado

## Manejo de errores

### Para funciones JavaScript

Las excepciones durante la evaluación del script se capturan y se finaliza la respuesta, pero no se establece un payload JSON de error automáticamente. Se recomienda que el propio script gestione códigos de estado y mensajes de error.

### Para funciones de base de datos

Los errores en funciones PL/pgSQL o PL/Python3U se propagan como parte de la ejecución SQL; la respuesta HTTP contendrá el resultado o fallará si la ejecución lanza excepción.

## Configuración CORS

El endpoint incluye configuración CORS completa:

```java
// OPTIONS en HttpVerticle para /functions
headers.add("Access-Control-Allow-Methods", "GET, POST, OPTIONS");
headers.add("Access-Control-Allow-Headers", "*");

// Respuestas PL en FunctionsHandler (DB)
headers.add("Access-Control-Allow-Methods", "GET, POST, DELETE, OPTIONS, HEAD");
headers.add("Access-Control-Allow-Headers", "X-Apollo-Tracing, Authorization, Cache-Control, X-XSRF, Origin, X-Requested-With, Content-Type, Accept, Content-Length");
```

## Casos de uso

### 1. Funciones de integración (JavaScript)
- Integración con APIs externas
- Procesamiento de datos complejos
- Manipulación de respuestas HTTP

### 2. Funciones de negocio (PL/pgSQL)
- Lógica de negocio compleja
- Operaciones transaccionales
- Cálculos matemáticos

### 3. Funciones de análisis (PL/Python3U)
- Análisis de datos con librerías Python
- Machine Learning
- Procesamiento de texto avanzado

## Consideraciones de rendimiento

1. **JavaScript**: Ejecución en memoria (JVM), ideal para operaciones rápidas; formateo de respuesta a cargo del script
2. **PL/pgSQL**: Ejecución en base de datos, optimizada para operaciones SQL
3. **PL/Python3U**: Ejecución híbrida, balance entre flexibilidad y rendimiento

## Notas sobre desarrollo

- Las funciones JavaScript deben finalizar la respuesta HTTP; no hay envoltorio automático del retorno.
- Para funciones PL personalizadas, el orden y tipo de `customParameters` determinan el binding.

## Seguridad

- Validación de parámetros por tipo
- Sanitización de entrada
- Control de acceso basado en contexto de usuario
- Manejo seguro de conexiones de base de datos

## Conclusión

El endpoint `/functions/` proporciona una arquitectura flexible y robusta para la ejecución de funciones almacenadas, soportando múltiples lenguajes de programación y ofreciendo diferentes niveles de integración con el sistema HTTP y la base de datos. 

### Comportamiento de JavaScript

Actualmente, los scripts controlan completamente la respuesta HTTP (status, headers y body). No existe modo de captura automática del valor de retorno.

### Comportamiento específico de Python (PL/Python3U)

Las funciones Python tienen características únicas:

- **Formato de respuesta fijo**: Siempre retornan resultados en formato array `[{"result": "valor"}]`
- **Ejecución como función de base de datos**: Se procesan a través de la capa de base de datos, no como scripts independientes
- **Sin flexibilidad de formato**: A diferencia de JavaScript, no pueden escribir directamente a la respuesta HTTP
- **Consistencia garantizada**: El formato de respuesta es siempre el mismo, proporcionando predictibilidad

### Consistencia y flexibilidad

La gestión unificada de resultados y el manejo de errores garantizan una experiencia consistente independientemente del lenguaje utilizado. El sistema proporciona diferentes niveles de flexibilidad:

#### JavaScript - Máxima flexibilidad
- **Consistencia**: Formato de respuesta uniforme con funciones de base de datos (en modo return value)
- **Flexibilidad**: Elección entre simplicidad (return values) y control (direct response)
- **Robustez**: Manejo mejorado de errores y casos edge como la respuesta "ok"
- **Facilidad de uso**: Guías completas de testing y debugging

#### Python (PL/Python3U) - Consistencia garantizada
- **Predictibilidad**: Siempre el mismo formato de respuesta array
- **Simplicidad**: No hay decisiones de formato que tomar
- **Confiabilidad**: Ejecución a través de la capa de base de datos garantiza consistencia

#### PL/pgSQL - Rendimiento optimizado
- **Eficiencia**: Ejecución nativa en PostgreSQL
- **Consistencia**: Mismo formato de respuesta que Python
- **Robustez**: Manejo transaccional integrado

