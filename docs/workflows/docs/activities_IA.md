# Actividades de IA (AI_AGENT_TASK)

## Documentación detallada del flujo

### Fase 1: inicialización del proceso y creación de actividades

#### 1.1 Creación de instancia de proceso
Cuando se inicia un proceso de workflow (manual, automático o vía API), se crean las actividades que lo componen, incluyendo las actividades de tipo `AI_AGENT_TASK`:

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
            .assignee(null)                           // Las actividades de IA no requieren asignación
            .entityId(entityId.orElse(null))         // Puede vincularse a entidad existente
            .processInstance(masterProcess.getId())
            .status(ActivityInstanceStatusEnum.PENDING.toString())
            .build();
        
        instancesApi.createActivityInstance(activityInstance, processInstancesDto.getId());
    });
}
```

**Puntos clave:**
- **Todas las actividades se crean por adelantado**: Las actividades de IA se crean cuando inicia el proceso
- **Estado inicial**: Todas las actividades comienzan como `PENDING`
- **Sin asignación requerida**: Las actividades de IA no requieren `assignee` ya que son automatizadas

#### 1.2 Configuración mínima requerida para actividades de IA
Para crear una actividad de IA se requiere la siguiente configuración mínima:

```sql
-- Función SQL para crear actividades de IA
CREATE OR REPLACE FUNCTION "Workflows"."createActivity"(
    activityname text, 
    processid integer, 
    activitytype text, 
    activitydescription text, 
    usersassigned text, 
    groupsassigned text, 
    entityname text, 
    functionname text,           -- NO USADO para AI_AGENT_TASK
    functionvariables text       -- NO USADO para AI_AGENT_TASK
)

-- Para actividades AI_AGENT_TASK se almacenan usando el ELSE branch:
INSERT INTO "Workflows"."Activity" (
    name, 
    process, 
    type, 
    description,
    agent,                       -- REQUERIDO: Nombre del agente de IA
    "agentPrompt"               -- REQUERIDO: Prompt para el agente
)
VALUES (
    activityname, 
    processid, 
    activitytype::"Workflows"."ActivityTypeEnum", 
    activitydescription
);
```

**Configuración mínima:**
- **Activity Name**: Nombre identificativo de la actividad
- **Agent**: Nombre del agente de IA que procesará la tarea
- **Agent Prompt**: Prompt/instrucciones para el agente de IA

#### 1.3 Almacenamiento en base de datos
Las instancias de actividad de IA se almacenan en la misma tabla `ActivityInstance` que las otras actividades:

```sql
CREATE TABLE "Workflows"."Activity" (
    agent text NULL,                     -- Nombre del agente de IA
    "agentPrompt" text NULL,             -- Prompt para el agente (máx 10,000 caracteres)
    description text NULL,
    "dueDate" timestamp NULL,
    entity text NULL,
    "entityMode" "Workflows"."entityModeList" NULL,
    "entitySelectionFunction" text NULL,
    "function" text NULL,                -- NO USADO para AI_AGENT_TASK
    "functionVariables" text NULL,       -- NO USADO para AI_AGENT_TASK
    id serial4 NOT NULL,
    "name" text NOT NULL,
    process int4 NOT NULL,
    "type" "Workflows"."ActivityTypeEnum" NOT NULL,
    "uniqueId" text NULL,
    
    -- Constraints importantes para AI
    CONSTRAINT "agentPrompt_check" CHECK ((length("agentPrompt") <= 10000)),
    CONSTRAINT agent_check CHECK ((length(agent) <= 1000))
);

CREATE TABLE "Workflows"."ActivityInstance" (
    id serial4 PRIMARY KEY,
    activity int4 NOT NULL,              -- Vincula a definición de Actividad
    assignee text NULL,                  -- NULL para actividades de IA
    "entityId" int4 NULL,                -- Registro de entidad en el que se trabaja
    "processInstance" int4 NOT NULL,     -- Vincula a ProcessInstance
    status "ActivityStateEnum" DEFAULT 'PENDING',
    "logActivity" text NULL,             -- Log de ejecución de IA (máx 5,000 caracteres)
    "result" text NULL                   -- Resultado del agente de IA (máx 2,000 caracteres)
);
```

### Fase 2: evaluación continua del workflow

#### 2.1 Motor de evaluación periódica de actividades
El motor de workflow evalúa continuamente todos los procesos activos:

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

#### 2.2 Transiciones de estado de actividad de IA
El sistema gestiona las actividades de IA de manera automatizada:

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

### Fase 3: ejecución automática de actividades de IA

#### 3.1 Inicio de actividad de IA
Cuando una actividad de IA pasa de `PENDING` a `IN_PROGRESS`, se ejecuta automáticamente:

```java
private void startActivity(ActivityInstanceDto activityInstanceDto) {
    log.debug("Starting activity: {}", activityInstanceDto);
    instancesApi.openActivityInstance(activityInstanceDto.getId());
    
    switch (activityInstanceDto.getActivityDetail().getType()) {
        case "AI_AGENT_TASK":
            InstancesModule.getAiExecutorService()
                .execute(activityInstanceDto.getProcessInstance() + "-" + activityInstanceDto.getId(),
                    activityInstanceDto);
            break;
        // ... otros tipos de actividad
    }
}
```

#### 3.2 Procesamiento de parámetros del agente de IA
Los parámetros del agente se procesan antes de la ejecución:

```java
public void execute(String taskId, ActivityInstanceDto activityInstanceDto) {
    // 1. Configurar la función de IA fija
    activityInstanceDto.getActivityDetail().setFunction("aia.invokeAIAgent");
    
    // 2. Extraer configuración de la actividad
    String agentName = activityInstanceDto.getActivityDetail().getAgent();
    String agentPrompt = activityInstanceDto.getActivityDetail().getAgentPrompt();
    String[] ids = taskId.split("-");
    String processInstanceId = ids[0];
    String activityInstanceId = ids[1];

    // 3. Enriquecer prompt con contexto del workflow
    String promptEnrichment = agentPrompt + String.format(
        "\n You must consider that both the process instance with id %s and the activity instance with id %s are in your context",
        processInstanceId, activityInstanceId);
    
    log.debug("Ejecutando AI Task en el agente: \"{}\" con prompt: \"{}\"", agentName, promptEnrichment);
    
    // 4. Construir parámetros JSON para la función aia.invokeAIAgent
    JSONArray jsonArray = new JSONArray();
    JSONObject jsonObject = new JSONObject();
    jsonObject.put("name", "aia.invokeAIAgent.agentname");
    jsonObject.put("type", "text");
    jsonObject.put("value", agentName);
    jsonArray.put(jsonObject);
    
    jsonObject = new JSONObject();
    jsonObject.put("name", "aia.invokeAIAgent.prompt");
    jsonObject.put("type", "text");
    jsonObject.put("value", promptEnrichment);
    jsonArray.put(jsonObject);
    
    // 5. Establecer parámetros de función
    activityInstanceDto.getActivityDetail().setFunctionVariables(jsonArray.toString());
}
```

**Puntos clave del procesamiento:**
- **Función fija**: Todas las actividades de IA usan la función `aia.invokeAIAgent`
- **Enriquecimiento de prompt**: Se añade contexto del proceso e instancia de actividad
- **Parámetros estándar**: `agentname` y `prompt` se pasan como variables de función
- **Conversión a JSON**: Los parámetros se convierten al formato esperado por FunctionsApi

#### 3.3 Mecanismo de ejecución de IA
Las actividades de IA se ejecutan utilizando el mismo sistema de funciones que las actividades FUNCTION:

```java
try {
    // Utilizar FunctionsApi para ejecutar aia.invokeAIAgent
    functionsApi.invokeFunction(activityInstanceDto).onComplete(ar -> {
        if (ar.failed() || ar.result() == null) {
            Throwable cause = ar.cause();
            log.error("Error invoking AI Agent function", cause);
            instancesApi.updateLogActivityInstance(
                activityInstanceDto.getId(),
                cause.getMessage(),
                "Completed Task " + taskId + " with result: ERROR - cause: " + cause.getMessage()
            );
            return;
        }

        String resultInvoke = ar.result().toString();
        log.debug("Resultado de la tarea: {}", resultInvoke);
        
        // Procesar respuesta del agente de IA
        processAIResponse(resultInvoke, taskId, activityInstanceDto);
    });
}
```

### Fase 4: procesamiento de datos del agente de IA

#### 4.1 Estructura de respuesta del agente
El agente de IA debe devolver una respuesta en formato JSON específico:

```java
@Data
public class AIAgentResponse {
    private String status;    // "SUCCESS" o "ERROR"
    private String response;  // Respuesta del agente o mensaje de error
}
```

#### 4.2 Procesamiento de respuesta exitosa
Cuando el agente devuelve una respuesta válida:

```java
try {
    ObjectMapper mapper = new ObjectMapper();
    AIAgentResponse result = mapper.readValue(resultInvoke, new TypeReference<>() {});

    if (result == null || "ERROR".equalsIgnoreCase(result.getStatus())) {
        // Manejar error del agente
        log.error("AI Agent Task '{}' could not be completed due to '{}'",
            activityInstanceDto.getActivityDetail().getName(),
            result != null ? result.getResponse() : "Unknown error");
        instancesApi.updateLogActivityInstance(
            activityInstanceDto.getId(),
            "ERROR",
            "Completed Task " + taskId + " with result: ERROR - cause: " 
                + (result != null ? result.getResponse() : "Unknown error")
        );
        instancesApi.errorActivityInstance(activityInstanceDto.getId());
        return;
    }

    // Respuesta exitosa
    instancesApi.updateLogActivityInstance(
        activityInstanceDto.getId(),
        resultInvoke,
        "Completed Task " + taskId + " with result: " + resultInvoke
    );

    // Marcar actividad como completada
    instancesApi.completeActivityInstance(activityInstanceDto.getId());
    
} catch (Exception e) {
    log.error("Error parsing AI Agent response", e);
    instancesApi.updateLogActivityInstance(
        activityInstanceDto.getId(),
        e.getMessage(),
        "Completed Task " + taskId + " with result: ERROR - cause: " + e.getMessage()
    );
    instancesApi.errorActivityInstance(activityInstanceDto.getId());
}
```

#### 4.3 Estados de finalización
Las actividades de IA pueden finalizar en diferentes estados:

**Estado DONE (Exitoso):**
- El agente devuelve `status: "SUCCESS"`
- La respuesta se parsea correctamente como `AIAgentResponse`
- Se actualiza el log con el resultado completo
- La actividad se marca como `DONE`

**Estado ERROR (Fallido):**
- El agente devuelve `status: "ERROR"`
- Error en la comunicación con el agente (`ar.failed()`)
- Error al parsear la respuesta JSON
- La actividad se marca como `ERROR`

### Fase 5: manejo de resultados y finalización

#### 5.1 Diferencia clave con actividades de función
**IMPORTANTE**: Las actividades de IA **NO** crean variables de workflow automáticamente como las funciones.

```java
// Las funciones crean variables de workflow:
private void onResponse(AutomaticTaskResponse response) {
    if (response.getResult()) {
        List<VariableDto> variables = createVariables(response.getParameters(), processInstanceId);
        variableApi.createProcessInstanceVariables(variables);  // ✅ Crea variables
        instancesApi.completeActivityInstance(Integer.valueOf(activityInstanceId));
    }
}

// Las actividades de IA NO crean variables:
// Solo marcan la actividad como completada
instancesApi.completeActivityInstance(activityInstanceDto.getId());  // ❌ No crea variables
```

#### 5.2 Diferencia clave con actividades de función
**IMPORTANTE**: Las actividades de IA **NO** crean variables de workflow automáticamente como las funciones, pero sí almacenan sus resultados en logs para consulta posterior.

### Fase 6: disponibilidad de resultados para actividades posteriores

#### 6.1 Acceso limitado a resultados de IA
Los resultados de las actividades de IA están disponibles únicamente a través del log de actividad:

**Almacenamiento del resultado:**
```java
instancesApi.updateLogActivityInstance(
    activityInstanceDto.getId(),
    resultInvoke,                    // Resultado completo del agente
    "Completed Task " + taskId + " with result: " + resultInvoke
);
```

**Limitaciones de acceso:**
- **No hay variables de workflow**: Los resultados no se crean como variables del proceso
- **Solo en logs**: El resultado se almacena en el campo `logActivity` de la instancia
- **No referenciales**: No se pueden usar en condiciones JEXL de transiciones
- **Límite de tamaño**: Máximo 5,000 caracteres en `logActivity`

#### 6.2 Implicaciones para el diseño de workflow
Debido a las limitaciones de acceso a resultados:

**Uso recomendado:**
```
[Actividad Manual] → [Actividad IA] → [Actividad Manual]
                                    ↓
                              Resultado solo para
                              consulta/logging
```

**Uso NO recomendado:**
```
[Actividad IA] → [Condición basada en resultado] → [Siguiente actividad]
                        ❌ NO FUNCIONA
```

### Fase 7: manejo de errores

#### 7.1 Tipos de errores en actividades de IA

**Error de comunicación:**
```java
if (ar.failed() || ar.result() == null) {
    Throwable cause = ar.cause();
    log.error("Error invoking AI Agent function", cause);
    // Actividad marcada como ERROR
}
```

**Error del agente:**
```java
if (result == null || "ERROR".equalsIgnoreCase(result.getStatus())) {
    log.error("AI Agent Task '{}' could not be completed due to '{}'",
        activityInstanceDto.getActivityDetail().getName(),
        result != null ? result.getResponse() : "Unknown error");
    // Actividad marcada como ERROR
}
```

**Error de parsing:**
```java
catch (Exception e) {
    log.error("Error parsing AI Agent response", e);
    // Actividad marcada como ERROR
}
```

#### 7.2 Estados de error
Las actividades de IA pueden tener los siguientes estados de error:
- **ERROR**: El agente falló, hubo error de comunicación, o error de parsing
- **Log actualizado**: Se registra el mensaje de error para diagnóstico
- **Workflow bloqueado**: El proceso no puede continuar hasta resolver el error

### Fase 8: limitaciones y restricciones

#### 8.1 Limitaciones de base de datos
Restricciones importantes definidas en el esquema:

```sql
-- Límites de tamaño para actividades de IA
CONSTRAINT "agentPrompt_check" CHECK ((length("agentPrompt") <= 10000)),  -- Prompt máx 10KB
CONSTRAINT agent_check CHECK ((length(agent) <= 1000)),                  -- Nombre agente máx 1KB

-- Límites para instancias
CONSTRAINT "logActivity_check" CHECK ((length("logActivity") <= 5000)),   -- Log máx 5KB  
CONSTRAINT result_check CHECK ((length(result) <= 2000)),                -- Resultado máx 2KB
```

#### 8.2 Limitaciones funcionales

**Dependencia de función externa:**
- Todas las actividades de IA dependen de la función `aia.invokeAIAgent`
- Si esta función no existe o falla, todas las actividades de IA fallarán

**Sin variables de workflow:**
- Los resultados no se almacenan como variables del proceso
- No se pueden referenciar en condiciones de transición
- Limita la capacidad de crear workflows condicionales basados en IA

**Evaluación de finalización:**
```java
// Las actividades de IA NO tienen evaluación de finalización personalizada
return switch (activityInstanceDto.getActivityDetail().getType()) {
    case "AND" -> evaluateAndCompletion(...);
    case "OR" -> evaluateOrCompletion(...);
    case "XOR" -> evaluateXorCompletion(...);
    case "HUMAN_TASK" -> evaluateHumanTaskCompletion(...);
    default -> Future.succeededFuture(false);  // ❌ AI_AGENT_TASK usa default
};
```

**Finalización inmediata:**
- Las actividades de IA se marcan como completadas inmediatamente después de recibir respuesta
- No hay evaluación de condiciones como en actividades manuales
- No hay re-evaluación periódica

## Resumen del flujo completo

### Secuencia del flujo de datos para actividades de IA
1. **Inicio de proceso** → Todas las instancias de actividad creadas con estado `PENDING`
2. **Evaluación automática** → Motor de workflow detecta actividades de IA `PENDING` elegibles
3. **Inicio automático** → Actividad cambia a `IN_PROGRESS` y se ejecuta inmediatamente
4. **Configuración de parámetros** → Se establecen `agentname` y `prompt` enriquecido
5. **Ejecución vía función** → Se invoca `aia.invokeAIAgent` usando FunctionsApi
6. **Procesamiento de respuesta** → Se parsea `AIAgentResponse` del agente
7. **Finalización** → Estado cambia a `DONE` o `ERROR`, log actualizado
8. **Progresión del workflow** → Siguientes actividades activadas basadas en transiciones
9. **Sin variables** → Los resultados NO están disponibles como variables de workflow

### Transiciones de estado de actividad de IA
```
PENDING → IN_PROGRESS → DONE/ERROR
   ↓           ↓            ↓
Creada    Ejecutándose    Completada
           vía IA Agent   (sin variables)
```

### Diferencias clave con otras actividades

| Aspecto | Actividades Manuales | Actividades de Función | Actividades de IA |
|---------|---------------------|----------------------|-------------------|
| **Inicio** | Requiere asignación y acción del usuario | Automático cuando se cumplen condiciones | Automático cuando se cumplen condiciones |
| **Ejecución** | Formulario completado por usuario | Función ejecutada en BD | Agente de IA invocado vía función |
| **Parámetros** | Datos de entidad del formulario | Variables evaluadas del contexto | Agent name + prompt enriquecido |
| **Resultado** | Variable de workflow creada | Variables de workflow con resultado | Solo log de actividad |
| **Disponibilidad** | Variables disponibles para condiciones | Variables disponibles para condiciones | **NO disponible para condiciones** |
| **Tiempo** | Indeterminado (depende del usuario) | Inmediato (función local) | Depende del agente de IA externo |
| **Evaluación** | Evaluación de finalización compleja | Finalización automática | Finalización inmediata |

### Puntos clave de integración

#### Configuración de actividad de IA
- **Agent**: Nombre del agente de IA (máx 1,000 caracteres)
- **Agent Prompt**: Instrucciones para el agente (máx 10,000 caracteres)
- **Función fija**: Siempre usa `aia.invokeAIAgent`
- **Sin parámetros adicionales**: No utiliza `functionVariables` del usuario

#### Dependencias externas
- **Función requerida**: `aia.invokeAIAgent` debe existir en la base de datos
- **Agente de IA**: El agente especificado debe estar disponible y funcional
- **Formato de respuesta**: El agente debe devolver JSON con `status` y `response`

#### Limitaciones importantes
- **Sin variables de workflow**: Los resultados no se almacenan como variables
- **Sin condicionales**: No se pueden usar resultados en condiciones de transición
- **Límites de tamaño**: Restricciones estrictas en prompt, logs y resultados
- **Dependencia externa**: Falla si el agente de IA no está disponible

#### Casos de uso recomendados
- **Procesamiento de documentos**: Análisis o extracción de información
- **Clasificación**: Categorización automática de contenido
- **Generación de contenido**: Creación de textos o respuestas
- **Logging inteligente**: Registro de decisiones o análisis para auditoría
- **Workflows lineales**: Donde el resultado de IA no afecta el flujo condicional
