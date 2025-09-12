# Análisis del endpoint `/functions/` - Ejecución de funciones almacenadas en base de datos

## Introducción

El endpoint `/functions/` de Airflows es un sistema robusto que permite la ejecución programática de funciones almacenadas en base de datos. Este endpoint soporta tres tipos de lenguajes de programación diferentes y maneja la ejecución, el procesamiento de parámetros y la gestión de resultados de manera unificada.

## Arquitectura del endpoint

### Configuración de rutas

El endpoint está configurado en `com.niledb.platform/src/main/java/verticles/HttpVerticle.java` con las siguientes rutas:

```java
// Líneas 231-233 en HttpVerticle.java
router.get("/functions/:functionName").blockingHandler(FunctionsHandler::execute);
router.post("/functions/:functionName").handler(BodyHandler.create())
    .blockingHandler(FunctionsHandler::execute);
router.options("/functions/:functionName").blockingHandler(routingContext -> {
    // Configuración CORS
});
```

### Componentes principales

1. **FunctionsHandler**: Maneja las peticiones HTTP y coordina la ejecución
2. **FunctionsHelper**: Gestiona la carga y configuración de funciones
3. **FunctionExecutor**: Ejecuta funciones directamente sin overhead HTTP
4. **CustomRepository**: Ejecuta funciones de base de datos

## Lenguajes soportados

### 1. ECMAScriptNashorn (JavaScript)

**Características:**
- Motor de JavaScript Nashorn integrado en la JVM
- Acceso completo al contexto HTTP (request/response)
- Ejecución directa en el servidor
- **Nuevo**: Soporte para captura de valores de retorno
- **Nuevo**: Dos modos de ejecución (return value y direct response)

**Implementación:**
```java
// En com.niledb.platform/src/main/java/helpers/FunctionsHelper.java (líneas 87-88)
engine = new ScriptEngineManager().getEngineByName("nashorn");

// En com.niledb.platform/src/main/java/helpers/FunctionsHandler.java (líneas 55-58)
ScriptContext context = new SimpleScriptContext();
context.setAttribute("routingContext", new SafeRoutingContext(routingContext), ScriptContext.ENGINE_SCOPE);
FunctionsHelper.engine.eval(FunctionsHelper.functions.get(functionName), context);
```

#### Modos de ejecución JavaScript

El sistema JavaScript ahora soporta dos modos de ejecución que se detectan automáticamente:

##### 1. Modo Return Value (Nuevo)
Cuando la función retorna un valor no nulo y no vacío:

```javascript
function myFunction() {
    var request = routingContext.request();
    var param = request.getParam("name");
    
    // Procesar el parámetro
    var result = "Hello, " + param + "!";
    
    // Retornar el resultado (será capturado y formateado como JSON)
    return result;
}
```

**Respuesta HTTP:**
```json
[
  {
    "result": "Hello, John!"
  }
]
```

##### 2. Modo Direct Response (Legacy)
Cuando la función escribe directamente a la respuesta HTTP:

```javascript
function myLegacyFunction() {
    var request = routingContext.request();
    var param = request.getParam("name");
    
    // Escribir directamente a la respuesta (comportamiento legacy)
    routingContext.response()
        .putHeader("Content-Type", "application/json")
        .end(JSON.stringify({message: "Hello, " + param + "!"}));
}
```

**Respuesta HTTP:**
```json
{"message": "Hello, John!"}
```

#### Detección automática de modo

El sistema detecta automáticamente qué modo usar basándose en si la función JavaScript retorna un valor:

```java
// En com.niledb.platform/src/main/java/helpers/FunctionsHelper.java (líneas 374-382)
if (result != null && !result.toString().isEmpty()) {
    // La función retornó un valor - usar modo return value
    return new JavaScriptFunctionResult(result.toString(), ExecutionMode.RETURN_VALUE);
} else {
    // La función escribió directamente a la respuesta - usar modo direct response
    return new JavaScriptFunctionResult("", ExecutionMode.DIRECT_RESPONSE);
}
```

#### Análisis del problema "ok" response

**Problema común:** Cuando una función JavaScript retorna `undefined` o no retorna nada, el sistema genera una respuesta "ok" por defecto.

**Causa raíz:**
1. La función retorna `undefined` (explícitamente o implícitamente)
2. El sistema detecta que no hay valor de retorno válido
3. Se activa el modo DIRECT_RESPONSE
4. Como la función no escribió a la respuesta, se envía el valor por defecto "ok"

**Ejemplo problemático:**
```javascript
function execute(){
    var foo = 2 + 1
    return undefined  // Esto causa la respuesta "ok"
}
```

**Soluciones:**

1. **Retornar un valor significativo:**
```javascript
function execute(){
    var foo = 2 + 1
    return foo;  // Retornar el valor real en lugar de undefined
}
```

2. **Escribir directamente a la respuesta:**
```javascript
function execute(){
    var foo = 2 + 1
    routingContext.response().end(JSON.stringify({result: foo}));
}
```

3. **Retornar cualquier string no vacío:**
```javascript
function execute(){
    var foo = 2 + 1
    return "success";  // Cualquier string no vacío funcionará
}
```

#### Componentes técnicos nuevos

##### JavaScriptFunctionExecutor
```java
public class JavaScriptFunctionExecutor {
    public static JavaScriptFunctionResult execute(String functionName, RoutingContext routingContext);
    public static void formatResponse(JavaScriptFunctionResult result, HttpServerResponse response);
}
```

##### Modos de ejecución
```java
public enum ExecutionMode {
    RETURN_VALUE,    // La función retornó un valor
    DIRECT_RESPONSE, // La función escribió directamente a la respuesta
    ERROR           // La ejecución de la función falló
}
```

#### Logs de debugging

El sistema ahora incluye logs detallados para debugging:

```
JavaScript function MySchema.myFunction executed. Result: null, Type: null
Using DIRECT_RESPONSE mode for function MySchema.myFunction
Using DIRECT_RESPONSE mode - function already wrote to response
Function didn't write to response, sending default 'ok' response
```

#### Guía de testing

##### Test 1: Función que retorna un valor
```javascript
function myTestFunction() {
    var request = routingContext.request();
    var param = request.getParam("test");
    
    // Retornar un valor (esto debería ser capturado)
    return "Hello, " + param + "!";
}
```

**Respuesta esperada:**
```json
[
  {
    "result": "Hello, world!"
  }
]
```

##### Test 2: Función que escribe a la respuesta (legacy)
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

##### Test 3: Función que no hace nada (causa respuesta "ok")
```javascript
function myBrokenFunction() {
    var request = routingContext.request();
    var param = request.getParam("test");
    
    // Esta función no hace nada - no retorna, no escribe a la respuesta
    // Esto causará la respuesta "ok"
}
```

**Respuesta esperada:**
```
ok
```

#### Problemas comunes y soluciones

##### Problema 1: La función no retorna nada
**Problema:** Tu función procesa datos pero no los retorna
**Solución:** Agregar una declaración `return`
```javascript
function processData() {
    var data = "processed";
    // Faltante: return data;
}
```

##### Problema 2: La función retorna undefined
**Problema:** La función JavaScript retorna `undefined`
**Solución:** Asegurarse de que la declaración return sea alcanzada
```javascript
function mightReturnUndefined() {
    if (someCondition) {
        return "value";
    }
    // Esto retorna undefined si la condición es falsa
}
```

##### Problema 3: La función escribe a la respuesta pero también retorna
**Problema:** La función hace ambas cosas, causando confusión
**Solución:** Elegir un enfoque - o retornar un valor O escribir a la respuesta

#### Mejores prácticas

1. **Usar valores de retorno** para procesamiento simple de datos
2. **Usar respuesta directa** para headers personalizados o respuestas complejas
3. **No mezclar ambos enfoques** en la misma función
4. **Siempre probar las funciones** con los logs de debugging habilitados

#### Migración y compatibilidad

- **Funciones existentes:** Las funciones JavaScript que escriben directamente a la respuesta continuarán funcionando sin cambios (modo direct response)
- **Nuevas funciones:** Puedes elegir cualquier enfoque:
  - **Modo return value:** Más simple, más consistente con funciones de base de datos
  - **Modo direct response:** Más control sobre formato de respuesta y headers

### 2. PL/pgSQL

**Características:**
- Lenguaje procedural nativo de PostgreSQL
- Ejecución directa en la base de datos
- Soporte para parámetros tipados

**Implementación:**
```java
// En com.niledb.platform/src/main/java/workflows/db/repository/CustomRepository.java (líneas 42-58)
String sql = String.format("select \"%s\".\"%s\" (%s)", schema, function, params);
return client.preparedQuery(sql).execute()
    .map(rows -> {
        RowIterator<Row> it = rows.iterator();
        if (it.hasNext()) {
            return it.next().getValue(0);
        }
        return null;
    });
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
// En com.niledb.platform/src/main/java/helpers/FunctionsHandler.java (líneas 134-136)
PreparedStatement ps = connection.prepareStatement(
    "SELECT \"" + functionName.split("\\.")[0] + "\".\"" + functionName.split("\\.")[1] + "\"(" + 
    (parameterNames != null ? String.join(", ", "?".repeat(parameterNames.length).split("")) : "") + 
    ") as \"result\"");
```

#### Procesamiento de resultados Python

El sistema procesa el `ResultSet` y convierte cada fila a un objeto JSON:

```java
// En com.niledb.platform/src/main/java/helpers/FunctionsHandler.java (líneas 187-199)
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

Las funciones JavaScript tienen más flexibilidad:
- **Modo return value**: Pueden retornar valores que se formatean como arrays (similar a Python)
- **Modo direct response**: Pueden escribir directamente a la respuesta HTTP con formato personalizado

Pero las funciones Python siempre se procesan a través de la capa de base de datos, lo que impone el formato de array.

**Conclusión**: Las funciones Python en el endpoint `/functions/` siempre retornarán sus resultados envueltos en una estructura de array JSON.

## Procesamiento de parámetros

### Para funciones JavaScript (ECMAScriptNashorn)

Los parámetros se pasan a través del contexto HTTP con tres métodos diferentes:

#### 1. Query parameters
```javascript
function myFunction() {
    var request = routingContext.request();
    var param = request.getParam("parameterName");
    // Usar el parámetro
    return "Hello, " + param + "!";
}
```

**Uso:** `GET /functions/myFunction?parameterName=value`

#### 2. HTTP headers
```javascript
function myFunction() {
    var request = routingContext.request();
    var param = request.getHeader("parameterName");
    // Usar el parámetro
    return "Hello, " + param + "!";
}
```

**Uso:** Incluir header `parameterName: value` en la petición HTTP

#### 3. JSON body (para peticiones POST)
```javascript
function myFunction() {
    var body = routingContext.body().asJsonObject();
    var param = body.getString("parameterName");
    // Usar el parámetro
    return "Hello, " + param + "!";
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
    
    // Obtener de query parameters
    var queryParam = request.getParam("queryParam");
    
    // Obtener de headers
    var headerParam = request.getHeader("headerParam");
    
    // Obtener de JSON body
    var bodyParam = body.getString("bodyParam");
    
    // Procesar los datos
    var result = {
        query: queryParam,
        header: headerParam,
        body: bodyParam
    };
    
    return result;
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

#### Implementación técnica del procesamiento de parámetros
```java
// En com.niledb.platform/src/main/java/helpers/FunctionsHandler.java (líneas 71-99)
// Parsing de parámetros personalizados
String[] parameters = FunctionsHelper.plCustomFunctionsParameters.get(functionName).split(",");
for (int i = 0; i < parameters.length; i++) {
    String paramName = parameters[i].split("\\s+")[0];
    String paramType = parameters[i].split("\\s+")[1];
    
    // Mapeo de tipos (líneas 134-153)
    switch (paramType) {
        case "text": ps.setObject(i + 1, parameterValues[i], Types.VARCHAR); break;
        case "int": ps.setObject(i + 1, parameterValues[i], Types.INTEGER); break;
        case "numeric": ps.setObject(i + 1, parameterValues[i], Types.NUMERIC); break;
        case "boolean": ps.setObject(i + 1, parameterValues[i], Types.BOOLEAN); break;
        case "date": ps.setObject(i + 1, parameterValues[i], Types.DATE); break;
    }
}
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

**Procesamiento de resultados:**

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
// En com.niledb.platform/src/main/java/helpers/FunctionsHandler.java (línea 201)
response.end(jsonArray.encodePrettily());
```

### Formato de respuesta estándar

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
| **Flexibilidad de formato** | Alta - dos modos de ejecución | Baja - siempre formato array |
| **Modo return value** | ✅ Opcional | ✅ Siempre aplicado |
| **Modo direct response** | ✅ Disponible | ❌ No disponible |
| **Control de headers** | ✅ Completo | ❌ Limitado |
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
-- En com.niledb.platform/src/main/java/helpers/FunctionsHelper.java (línea 118)
-- Para JavaScript
SELECT schema, name, contents, "cronExpression", "isHttpEnabled", 
       "interceptHttpRequests", "interceptSelects" 
FROM "Models"."Function" 
WHERE language = 'ECMAScriptNashorn'

-- En com.niledb.platform/src/main/java/helpers/FunctionsHelper.java (línea 151)
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

#### Manejo de errores mejorado
```java
// En com.niledb.platform/src/main/java/workflows/usecase/FunctionExecutor.java (líneas 77-91)
try {
    FunctionsHelper.engine.eval(FunctionsHelper.functions.get(functionName), context);
} catch (Exception ex) {
    if (routingContext != null) {
        routingContext.response().setStatusCode(500).end("Error executing Javascript function");
    }
    return Future.failedFuture("Error executing Javascript function");
}
```

#### Nuevo sistema de manejo de errores
```java
// En com.niledb.platform/src/main/java/helpers/FunctionsHelper.java
// Los errores son capturados y retornados como respuestas HTTP 500
{
  "error": "Error executing JavaScript function: [detalles del error]"
}
```

#### Logs de debugging para errores
```
JavaScript function MySchema.myFunction executed. Result: null, Type: null
Using DIRECT_RESPONSE mode for function MySchema.myFunction
Function didn't write to response, sending default 'ok' response
```

#### Casos de error comunes
1. **Función retorna undefined**: Genera respuesta "ok" por defecto
2. **Función no retorna nada**: Genera respuesta "ok" por defecto  
3. **Error de sintaxis JavaScript**: Capturado y retornado como error HTTP 500
4. **Error de ejecución**: Capturado y retornado como error HTTP 500

### Para funciones de base de datos

```java
// En com.niledb.platform/src/main/java/workflows/usecase/FunctionExecutor.java (líneas 121-124)
try {
    // Ejecución de la función
} catch (Exception ex) {
    log.error("Error executing database function {}: {}", functionName, ex.getMessage(), ex);
    return Future.succeededFuture("error: " + ex.getMessage());
}
```

## Configuración CORS

El endpoint incluye configuración CORS completa:

```java
// En com.niledb.platform/src/main/java/helpers/FunctionsHandler.java (líneas 113-123)
String origin = request.headers().get("Origin");
if (origin != null && !origin.equals("")) {
    headers.add("Access-Control-Allow-Origin", origin);
    headers.add("Access-Control-Allow-Credentials", "true");
    headers.add("Vary", "Accept-Encoding, Origin");
} else {
    headers.add("Access-Control-Allow-Origin", "*");
}
headers.add("Access-Control-Allow-Methods", "GET, POST, DELETE, OPTIONS, HEAD");
headers.add("Access-Control-Allow-Headers", 
    "X-Apollo-Tracing, Authorization, Cache-Control, X-XSRF, Origin, " +
    "X-Requested-With, Content-Type, Accept, Content-Length");
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

1. **JavaScript**: Ejecución en memoria, ideal para operaciones rápidas
   - **Nuevo**: Captura de valores de retorno sin overhead adicional
   - **Nuevo**: Detección automática de modo de ejecución optimizada
2. **PL/pgSQL**: Ejecución en base de datos, optimizada para operaciones SQL
3. **PL/Python3U**: Ejecución híbrida, balance entre flexibilidad y rendimiento

## Beneficios del nuevo sistema JavaScript

### 1. Consistencia
- Las funciones JavaScript ahora pueden retornar valores como las funciones de base de datos
- Formato de respuesta uniforme: `[{"result": "valor"}]`
- API consistente entre todos los tipos de funciones

### 2. Compatibilidad hacia atrás
- Las funciones existentes que escriben directamente a la respuesta continúan funcionando
- No se requieren cambios en el código existente
- Migración gradual posible

### 3. Flexibilidad
- Elegir el enfoque que mejor se adapte al caso de uso
- **Return value mode**: Más simple para procesamiento de datos
- **Direct response mode**: Más control para respuestas complejas

### 4. Manejo de errores mejorado
- Captura y reporte de errores mejorado
- Logs de debugging detallados
- Respuestas de error estructuradas

### 5. Experiencia de desarrollo
- Debugging más fácil con logs detallados
- Testing más simple con casos de prueba claros
- Documentación completa de problemas comunes

## Seguridad

- Validación de parámetros por tipo
- Sanitización de entrada
- Control de acceso basado en contexto de usuario
- Manejo seguro de conexiones de base de datos

## Conclusión

El endpoint `/functions/` proporciona una arquitectura flexible y robusta para la ejecución de funciones almacenadas, soportando múltiples lenguajes de programación y ofreciendo diferentes niveles de integración con el sistema HTTP y la base de datos. 

### Mejoras significativas en JavaScript

El sistema JavaScript ha sido significativamente mejorado con:

- **Captura de valores de retorno**: Las funciones JavaScript ahora pueden retornar valores que se formatean automáticamente como JSON
- **Dos modos de ejecución**: Return value mode (nuevo) y Direct response mode (legacy)
- **Detección automática**: El sistema detecta automáticamente qué modo usar
- **Debugging mejorado**: Logs detallados para facilitar el desarrollo y troubleshooting
- **Compatibilidad total**: Las funciones existentes continúan funcionando sin cambios

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

