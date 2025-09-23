# Actividades de funciones (FUNCTION)

## Documentación detallada del flujo

### Fase 1: inicialización del proceso y creación de actividades

#### 1.1 Creación de instancia de proceso
Cuando se inicia un proceso de workflow (manual, automático o vía API), se crean las actividades que lo componen, incluyendo las actividades de tipo `FUNCTION`:

```java
// StartProcessUseCase crea todas las instancias de actividad por adelantado
public boolean execute(ProcessInstanceDto processInstancesDto, ProcessDto masterProcess) {
    // Actualizar estado del proceso a IN_PROGRESS
    instancesApi.updateStatusProcessInstance(processInstancesDto.getId(), 
        ProcessStateEnum.IN_PROGRESS.toString());
    
    // Crear todas las actividades para este proceso
    masterProcess.getActivities().forEach(activity -> {
        var activityInstance = ActivityInstanceDto.builder()
            .activity(activity.getId())
            .assignee(null)                           // Las funciones no requieren asignación
            .entityId(entityId.orElse(null))         // Puede vincularse a entidad existente
            .processInstance(masterProcess.getId())
            .status(ActivityInstanceStatusEnum.PENDING.toString())
            .build();
        
        instancesApi.createActivityInstance(activityInstance, processInstancesDto.getId());
    });
}
```

**Puntos clave:**
- **Todas las actividades se crean por adelantado**: Las actividades de función se crean cuando inicia el proceso
- **Estado inicial**: Todas las actividades comienzan como `PENDING`
- **Sin asignación requerida**: Las actividades de función no requieren `assignee` ya que son automatizadas

#### 1.2 Configuración mínima requerida para actividades de función
Para crear una actividad de función se requiere la siguiente configuración mínima:

```sql
-- Función SQL para crear actividades de función
CREATE OR REPLACE FUNCTION "Workflows"."createActivity"(
    activityname text, 
    processid integer, 
    activitytype text, 
    activitydescription text, 
    usersassigned text, 
    groupsassigned text, 
    entityname text, 
    functionname text,           -- REQUERIDO para FUNCTION
    functionvariables text       -- JSON con parámetros de la función
)

-- Para actividades FUNCTION se almacenan:
INSERT INTO "Workflows"."Activity" (
    name, 
    process, 
    type, 
    description, 
    function,                    -- Nombre de la función a ejecutar
    "functionVariables"          -- Parámetros JSON
)
VALUES (
    activityname, 
    processid, 
    activitytype::"Workflows"."ActivityTypeEnum", 
    activitydescription, 
    functionname, 
    functionvariables
);
```

**Configuración mínima:**
- **Activity Name**: Nombre identificativo de la actividad
- **Function**: Identificador de la función a ejecutar (formato: `schema.functionName`)
- **Function Parameters**: JSON con los parámetros que se pasarán a la función

#### 1.3 Almacenamiento en base de datos
Las instancias de actividad de función se almacenan en la misma tabla `ActivityInstance` que las actividades manuales:

```sql
CREATE TABLE "Workflows"."ActivityInstance" (
    id serial4 PRIMARY KEY,
    activity int4 NOT NULL,              -- Vincula a definición de Actividad
    assignee text NULL,                  -- NULL para actividades de función
    "entityId" int4 NULL,                -- Registro de entidad en el que se trabaja
    "processInstance" int4 NOT NULL,     -- Vincula a ProcessInstance
    status "ActivityStateEnum" DEFAULT 'PENDING',
    "logActivity" text NULL,             -- Log de ejecución de función
    "result" text NULL                   -- Resultado de la función
);
```

### Fase 2: evaluación continua del workflow

#### 2.1 Motor de evaluación periódica de actividades
El motor de workflow evalúa continuamente todos los procesos activos cada 1 segundo:

```java
// PeriodicTasksVerticle se ejecuta periódicamente
private void scheduleNextWorkflowRun() {
    vertx.setTimer(DELAY_MS, id -> {
        runWorkflowExecutionTask().onComplete(res -> {
            if (res.succeeded()) {
                log.debug("Tarea completada correctamente.");
            } else {
                log.warn("Error en la tarea: {}", res.cause().getMessage());
            }
            scheduleNextWorkflowRun();
        });
    });
}

private Future<Void> runWorkflowExecutionTask() {
    return vertx.executeBlocking(() -> {
        try {
            InstancesModule.getInstancesService().runInstances();
        } catch (Exception e) {
            log.warn("Unexpected error in runWFInstances {}", e.getMessage());
        }
        return null;
    });
}
```

#### 2.2 Transiciones de estado de actividad de función
El sistema gestiona las actividades de función de manera automatizada:

```java
// EvaluateActivitiesUseCase maneja la progresión de actividades
switch (activityInstanceDto.getActivityDetail().getType()) {
    case "START" -> handleStartActivity();           // Punto de entrada del proceso
    case "HUMAN_TASK" -> evaluateHumanTaskCompletion(); // Actividades de tareas humanas
    case "FUNCTION" -> executeFunctionTask();        // Funciones automatizadas
    case "AI_AGENT_TASK" -> executeAiTask();        // Actividades impulsadas por IA
    case "AND", "OR", "XOR" -> evaluateLogicGates(); // Control de flujo
    case "END" -> handleEndActivity();              // Finalización del proceso
}
```

### Fase 3: ejecución automática de actividades de función

#### 3.1 Inicio de actividad de función
Cuando una actividad de función pasa de `PENDING` a `IN_PROGRESS`, se ejecuta automáticamente:

```java
private void startActivity(ActivityInstanceDto activityInstanceDto) {
    log.debug("Starting activity: {}", activityInstanceDto);
    instancesApi.openActivityInstance(activityInstanceDto.getId());
    
    switch (activityInstanceDto.getActivityDetail().getType()) {
        case "FUNCTION":
            InstancesModule.getFunctionsExecutorService()
                .execute(activityInstanceDto.getProcessInstance() + "-" + activityInstanceDto.getId(),
                    activityInstanceDto);
            break;
        // ... otros tipos de actividad
    }
}
```

#### 3.2 Procesamiento de parámetros de función
Los parámetros de la función se procesan antes de la ejecución:

```java
public Future<Object> invokeFunction(ActivityInstanceDto activityInstanceDto) {
    log.debug("Invoke Function from processInstanceId: {}",
        activityInstanceDto.getProcessInstance());

    // 1. Parsear variables de función desde JSON
    List<FunctionVariable> variables = parseFunctionVariables(activityInstanceDto);

    // 2. Evaluar variables con contexto del workflow
    return evaluateVariables(variables, activityInstanceDto)
        .flatMap(evaluatedVariables ->
            // 3. Ejecutar función con variables evaluadas
            customRepository.executeFunction(evaluatedVariables,
                activityInstanceDto.getActivityDetail().getFunction())
        );
}
```

**Estructura de FunctionVariable:**
```java
@Data
public class FunctionVariable {
    private String name;        // Nombre del parámetro
    private Object value;       // Valor del parámetro (puede ser referencia)
    private String type;        // Tipo de dato (int, text, etc.)
    private Integer order;      // Orden de los parámetros
}
```

#### 3.3 Evaluación de parámetros con contexto del workflow
Los parámetros pueden hacer referencia a datos del workflow:

```java
private Future<Void> evaluateVariable(
    FunctionVariable variable,
    List<VariableDto> instanceVariables,
    ActivityInstanceDto activityInstanceDto) {

    if (variable.getValue() instanceof String value && value.contains(".")) {
        String[] parts = value.split("\\.");

        if (parts.length == 3) {
            // Referencia a atributo de entidad: Application.Entity.Field
            return evaluateEntityVariable(variable, instanceVariables, parts);

        } else if (parts.length == 2) {
            if (parts[0].length() == 6) {
                // Referencia a actividad por uniqueId: uniqueId.field
                return instancesApi.fetchHumanActivityInstanceByUniqueId(
                        activityInstanceDto.getProcessInstanceDetail().getId().longValue(),
                        parts[0])
                    .compose(act -> {
                        // Obtener datos de entidad vinculada a la actividad
                        return entityApi.fetchAllAttributeValues(
                            act.getActivityDetail().getEntity(),
                            entityId,
                            List.of(parts[1])
                        ).compose(values -> {
                            if (!values.isEmpty()) {
                                variable.setValue(values.get(0).getValue());
                            }
                            return Future.succeededFuture((Void) null);
                        });
                    });
            } else {
                // Referencia a variable del proceso
                setVariableValue(variable, instanceVariables, value);
            }
        }
    }

    return Future.succeededFuture((Void) null);
}
```

**Tipos de referencias de parámetros:**
- **Variables del proceso**: `variableName` → Valor de variable de workflow
- **Campos de entidad**: `Application.Entity.Field` → Valor específico de atributo de entidad
- **Resultados de actividades**: `uniqueId.field` → Campo de entidad procesada por actividad específica
- **Resultados de funciones previas**: `Application.Function.result` → Resultado de función anterior

### Fase 4: procesamiento de datos según lenguaje de programación

#### 4.1 Lenguajes soportados
El sistema soporta múltiples lenguajes de programación para funciones:

```sql
-- Tipos de lenguaje soportados
CREATE TYPE "Models"."languageType" AS ENUM (
    'plpgsql',              -- PostgreSQL procedural language
    'ECMAScriptNashorn',    -- JavaScript (Nashorn engine)
    'plpython3u'            -- Python 3
);
```

#### 4.2 Ejecución de funciones PostgreSQL (plpgsql/plpython3u)
**Todas las funciones de base de datos (plpgsql y plpython3u) se ejecutan directamente en PostgreSQL** usando el mismo mecanismo:

```java
public Future<Object> executeFunction(List<FunctionVariable> variables, String functionName) {
    var nameVars = functionName.split("\\.");
    var schema = nameVars[0];
    var function = nameVars[1];

    // Construir parámetros ordenados por orden especificado
    String params = variables.isEmpty() ? "" : variables.stream()
        .sorted(Comparator.comparing(FunctionVariable::getOrder))
        .map(x -> {
            if ("int".equals(x.getType())) {
                return String.valueOf(x.getValue());
            } else {
                return "'" + String.valueOf(x.getValue()) + "'";
            }
        })
        .collect(Collectors.joining(","));

    // Ejecutar función SQL - funciona para plpgsql Y plpython3u
    String sql = String.format("select \"%s\".\"%s\" (%s)", schema, function, params);

    return client
        .preparedQuery(sql)
        .execute()
        .map(rows -> {
            RowIterator<Row> it = rows.iterator();
            if (it.hasNext()) {
                return it.next().getValue(0);  // Retornar resultado de función
            }
            return null;
        });
}
```

**Ejemplos de funciones PostgreSQL:**

*Función PL/pgSQL:*
```sql
CREATE OR REPLACE FUNCTION "MyApp"."calculateDiscount"(
    amount INTEGER, 
    percentage INTEGER
) RETURNS INTEGER AS $func$
BEGIN
    RETURN amount * percentage / 100;
END;
$func$ LANGUAGE plpgsql;
```

*Función Python (plpython3u):*
```sql
CREATE OR REPLACE FUNCTION "MyApp"."processData"(
    input_text TEXT,
    multiplier INTEGER
) RETURNS TEXT AS $func$
import json
import math

# Los parámetros están disponibles directamente
data = json.loads(input_text) if input_text else {}
result = {
    "processed": True,
    "length": len(input_text),
    "multiplied": len(input_text) * multiplier,
    "sqrt": math.sqrt(multiplier) if multiplier > 0 else 0
}

return json.dumps(result)
$func$ LANGUAGE plpython3u;
```

**Punto clave:** Las funciones Python se ejecutan **dentro de PostgreSQL** usando la extensión plpython3u, no como scripts Python independientes. Esto significa:
- Acceso completo a librerías Python estándar (json, math, datetime, etc.)
- Acceso a datos de base de datos mediante `plpy` module
- Ejecución en el mismo contexto transaccional que el workflow
- Retorno de valores directamente al flujo de trabajo

#### 4.3 Ejecución de funciones JavaScript (ECMAScriptNashorn)
**IMPORTANTE:** Las funciones JavaScript **NO** se ejecutan en workflows. De momento, las funciones JS se crean siempre como triggers.

Código relevante:

```java
// Este código se ejecuta solo para functiones HTTP
if (FunctionsHelper.functions.get(functionName) != null
    && FunctionsHelper.httpEnabledFunctions.get(functionName)) {
    ScriptContext context = new SimpleScriptContext();
    context.setAttribute("routingContext", new SafeRoutingContext(routingContext), ScriptContext.ENGINE_SCOPE);
    
    FunctionsHelper.engine.eval(FunctionsHelper.functions.get(functionName), context);
}
```

En BD:
```sql
-- Las funciones ECMAScriptNashorn se crean como triggers, NO como funciones ejecutables
IF function.language = 'ECMAScriptNashorn' THEN
    EXECUTE 'CREATE OR REPLACE FUNCTION "' || function.schema || '"."' || function.name || 
            '"() RETURNS trigger AS $func$DECLARE id integer;BEGIN ... END;$func$ LANGUAGE plpgsql';
```

**Conclusión:** Las funciones JavaScript (ECMAScriptNashorn) están diseñadas para:
- **Triggers de base de datos**: Se ejecutan automáticamente en respuesta a cambios en tablas
- **Endpoints HTTP**: Se pueden llamar directamente vía HTTP con `routingContext`
- **Interceptores**: Para interceptar requests HTTP o consultas GraphQL

**NO se pueden usar como funciones de workflow**. Solo las funciones `plpgsql` y `plpython3u` son compatibles con workflows.

#### 4.4 Diferencias en paso de parámetros por lenguaje

**PostgreSQL (plpgsql y plpython3u):**
- Los parámetros se pasan como argumentos posicionales ordenados
- Se construye la llamada SQL: `SELECT schema.function(param1, param2, ...)`
- Los tipos se manejan automáticamente: enteros sin comillas, strings con comillas
- **Python tiene acceso completo a librerías estándar y plpy module**

**JavaScript (ECMAScriptNashorn):**
- **NO DISPONIBLE para workflows**
- Solo para triggers, HTTP endpoints e interceptores
- En contextos HTTP: parámetros vía `routingContext` (request params, headers, body)

### Fase 5: manejo de resultados y finalización

#### 5.1 Procesamiento de resultado exitoso
Cuando la función se ejecuta exitosamente:

```java
private void handleSuccess(String taskId, ActivityInstanceDto activityInstanceDto, Object result) {
    log.debug("Resultado de la tarea: {}", result);
    
    // Crear variables de workflow con el resultado
    Map<String, Object> parameters = Map.of(
        activityInstanceDto.getActivityDetail().getUniqueId() + ".result", result,
        activityInstanceDto.getActivityDetail().getFunction() + ".result", result,
        taskId + ".result", result.toString()
    );

    // Actualizar log de actividad
    instancesApi.updateLogActivityInstance(
        activityInstanceDto.getId(),
        result.toString(),
        "Completed Task" + taskId + " with result: " + result
    );

    // Procesar respuesta
    onResponse(AutomaticTaskResponse.builder()
        .taskId(taskId)
        .parameters(parameters)
        .result(true)
        .build());
}
```

#### 5.2 Creación de variables de workflow
Los resultados se almacenan como variables de workflow para uso posterior:

```java
private void onResponse(AutomaticTaskResponse response) {
    String[] ids = response.getTaskId().split("-");
    String processInstanceId = ids[0];
    String activityInstanceId = ids[1];

    if (response.getResult()) {
        // Crear variables de workflow con los resultados
        List<VariableDto> variables = createVariables(response.getParameters(), processInstanceId);
        variableApi.createProcessInstanceVariables(variables);
        
        // Completar actividad
        instancesApi.completeActivityInstance(Integer.valueOf(activityInstanceId));
    } else {
        // Marcar actividad como error
        instancesApi.errorActivityInstance(Integer.valueOf(activityInstanceId));
    }
}

private List<VariableDto> createVariables(Map<String, Object> parameters, String processInstanceId) {
    List<VariableDto> variables = new ArrayList<>();
    parameters.forEach((key, value) -> {
        var builder = VariableDto.builder()
            .name(key)
            .description("FunctionVariable-" + key)
            .processInstance(Integer.valueOf(processInstanceId));

        // Mapeo de tipos automático
        if (value instanceof Date date) builder.type("DATE").valueDateTime(date);
        else if (value instanceof Integer i) builder.type("LONG").valueNumber(i);
        else if (value instanceof Double d) builder.type("DOUBLE").valueNumberWithDecimals(d);
        else if (value instanceof Boolean b) builder.type("BOOLEAN").valueBoolean(b);
        else builder.type("STRING").valueString(Objects.toString(value, ""));

        variables.add(builder.build());
    });
    return variables;
}
```

#### 5.3 Disponibilidad de resultados para actividades posteriores
Los resultados se hacen disponibles de múltiples formas:

**Nomenclatura de variables creadas:**
- `{uniqueId}.result` → Resultado por ID único de actividad (ej: `abc123.result`)
- `{functionName}.result` → Resultado por nombre de función (ej: `MyApp.calculateDiscount.result`)
- `{taskId}.result` → Resultado por ID de tarea (ej: `456-789.result`)

**Uso en condiciones de transición:**
```java
// Las condiciones pueden referenciar resultados de funciones
private Boolean evaluateCondition(String condition, Map<String, Object> variables) {
    try {
        // Evaluar condición JEXL con variables disponibles
        Object result = ees.evaluate(condition, variables);
        return result instanceof Boolean && (Boolean) result;
    } catch (Exception e) {
        log.error("Error evaluating condition '{}': {}", condition, e.getMessage());
        return false;
    }
}
```

**Ejemplos de uso:**
- Condición de transición: `MyApp_calculateDiscount_result > 100`
- Parámetro de función posterior: `MyApp.calculateDiscount.result`
- Referencia en actividad manual: Campo calculado disponible para formulario

### Fase 6: disponibilidad de resultados para actividades posteriores

#### 6.1 Creación de variables de workflow
Los resultados se almacenan como variables de workflow para uso posterior:

```java
private void onResponse(AutomaticTaskResponse response) {
    String[] ids = response.getTaskId().split("-");
    String processInstanceId = ids[0];
    String activityInstanceId = ids[1];

    if (response.getResult()) {
        // Crear variables de workflow con los resultados
        List<VariableDto> variables = createVariables(response.getParameters(), processInstanceId);
        variableApi.createProcessInstanceVariables(variables);
        
        // Completar actividad
        instancesApi.completeActivityInstance(Integer.valueOf(activityInstanceId));
    } else {
        // Marcar actividad como error
        instancesApi.errorActivityInstance(Integer.valueOf(activityInstanceId));
    }
}
```

#### 6.2 Nomenclatura de variables creadas
Los resultados se hacen disponibles de múltiples formas:

**Variables creadas:**
- `{uniqueId}.result` → Resultado por ID único de actividad (ej: `abc123.result`)
- `{functionName}.result` → Resultado por nombre de función (ej: `MyApp.calculateDiscount.result`)
- `{taskId}.result` → Resultado por ID de tarea (ej: `456-789.result`)

**Uso en condiciones de transición:**
```java
// Las condiciones pueden referenciar resultados de funciones
private Boolean evaluateCondition(String condition, Map<String, Object> variables) {
    try {
        // Evaluar condición JEXL con variables disponibles
        Object result = ees.evaluate(condition, variables);
        return result instanceof Boolean && (Boolean) result;
    } catch (Exception e) {
        log.error("Error evaluating condition '{}': {}", condition, e.getMessage());
        return false;
    }
}
```

### Fase 7: manejo de errores

#### 7.1 Procesamiento de errores de función
Cuando una función falla:

```java
private void handleFailure(Throwable t, String taskId) {
    String[] ids = taskId.split("-");
    String activityInstanceId = ids[1];
    
    log.error("Error activity {}", taskId, t);
    
    // Marcar actividad como error
    instancesApi.errorActivityInstance(Integer.valueOf(activityInstanceId));
    
    // Actualizar log con información del error
    instancesApi.updateLogActivityInstance(
        Integer.valueOf(activityInstanceId),
        t.getMessage(),
        "Error in Task" + taskId + " with result: " + t.getMessage()
    );
}
```

#### 7.2 Estados de error
Las actividades de función pueden tener los siguientes estados de error:
- **ERROR**: La función falló durante la ejecución
- **Log actualizado**: Se registra el mensaje de error para diagnóstico
- **Workflow bloqueado**: El proceso no puede continuar hasta resolver el error

## Resumen del flujo completo

### Secuencia del flujo de datos para actividades de función
1. **Inicio de proceso** → Todas las instancias de actividad creadas con estado `PENDING`
2. **Evaluación automática** → Motor de workflow detecta actividades de función `PENDING` elegibles
3. **Inicio automático** → Actividad cambia a `IN_PROGRESS` y se ejecuta inmediatamente
4. **Procesamiento de parámetros** → Variables JSON se evalúan con contexto del workflow
5. **Ejecución de función** → Se ejecuta según lenguaje (PostgreSQL, JavaScript, Python)
6. **Procesamiento de resultado** → Resultado se almacena como variables de workflow
7. **Finalización de actividad** → Estado cambia a `DONE`, log actualizado
8. **Progresión del workflow** → Siguientes actividades activadas basadas en transiciones
9. **Disponibilidad de resultados** → Resultados disponibles para condiciones y actividades posteriores

### Transiciones de estado de actividad de función
```
PENDING → IN_PROGRESS → DONE
   ↓           ↓          ↓
Creada    Ejecutándose  Completada
           Automática   con Resultado
```

### Diferencias clave con actividades manuales

| Aspecto | Actividades Manuales | Actividades de Función |
|---------|---------------------|----------------------|
| **Inicio** | Requiere asignación y acción del usuario | Automático cuando se cumplen condiciones |
| **Ejecución** | Formulario completado por usuario | Función ejecutada por sistema |
| **Parámetros** | Datos de entidad del formulario | Variables evaluadas del contexto del workflow |
| **Resultado** | Variable de workflow de entidad creada/actualizada | Variables de workflow con resultado de función |
| **Tiempo** | Indeterminado (depende del usuario) | Inmediato (limitado por tiempo de ejecución) |
| **Asignación** | Requiere `assignee` | No requiere asignación |

### Puntos clave de integración

#### Configuración de función
- **Identificador de función**: Formato `schema.functionName`
- **Parámetros JSON**: Array de `FunctionVariable` con name, value, type, order
- **Lenguajes soportados para workflows**: PostgreSQL (plpgsql), Python (plpython3u)
- **JavaScript (ECMAScriptNashorn)**: Solo para triggers, HTTP endpoints e interceptores - NO disponible para workflows

#### Evaluación de parámetros
- **Referencias de entidad**: `Application.Entity.Field`
- **Variables de proceso**: Nombre directo de variable
- **Resultados de actividades**: `uniqueId.field`
- **Resultados de funciones**: `Application.Function.result`

#### Resultados y continuación
- **Variables múltiples**: Se crean con diferentes nomenclaturas para flexibilidad
- **Condiciones de transición**: Pueden referenciar resultados usando expresiones JEXL
- **Disponibilidad inmediata**: Resultados disponibles tan pronto como la función completa
