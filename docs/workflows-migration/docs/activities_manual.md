# Actividades manuales (HUMAN_TASK)

## Documentación detallada del flujo

### Fase 1: inicialización del proceso y creación de actividades

#### 1.1 Creación de instancia de proceso
Cuando se inicia un proceso de workflow (manual, automático o vía API), se crean las actividades que lo componen:

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
            .assignee(null)                           // Inicialmente sin asignar
            .entityId(entityId.orElse(null))         // Puede vincularse a entidad existente
            .processInstance(masterProcess.getId())
            .status(ActivityInstanceStatusEnum.PENDING.toString())
            .build();
        
        instancesApi.createActivityInstance(activityInstance, processInstancesDto.getId());
    });
}
```

**Puntos clave:**
- **Todas las actividades se crean por adelantado**: Las actividades de tareas humanas se crean cuando inicia el proceso
- **Estado inicial**: Todas las actividades comienzan como `PENDING`
- **Sin asignación requerida**: Las actividades manuales requieren asignación posterior por parte del usuario

#### 1.2 Configuración mínima requerida para actividades manuales
Para crear una actividad manual se requiere la siguiente configuración mínima:

```sql
-- Función SQL para crear actividades manuales
CREATE OR REPLACE FUNCTION "Workflows"."createActivity"(
    activityname text, 
    processid integer, 
    activitytype text, 
    activitydescription text, 
    usersassigned text,          -- REQUERIDO: usuarios asignados
    groupsassigned text,         -- REQUERIDO: grupos asignados  
    entityname text,             -- REQUERIDO: entidad a crear/editar
    functionname text,           -- NO USADO para HUMAN_TASK
    functionvariables text       -- NO USADO para HUMAN_TASK
)

-- Para actividades HUMAN_TASK se almacenan:
INSERT INTO "Workflows"."Activity" (
    name, 
    process, 
    type, 
    description,
    entity,                      -- Entidad que se creará/editará
    "entityMode",               -- NEW o EXISTING
    "userAssigned",             -- Lista de usuarios autorizados
    "groupAssigned"             -- Lista de grupos autorizados
)
VALUES (
    activityname, 
    processid, 
    activitytype::"Workflows"."ActivityTypeEnum", 
    activitydescription,
    entityname,
    entitymode,
    usersassigned,
    groupsassigned
);
```

**Configuración mínima:**
- **Activity Name**: Nombre identificativo de la actividad
- **Entity**: Nombre de la entidad que se creará o editará
- **Entity Mode**: `NEW` (crear) o `EXISTING` (editar)
- **Users/Groups Assigned**: Usuarios y grupos autorizados para la actividad

#### 1.3 Almacenamiento en base de datos
Las instancias de actividad manual se almacenan en la misma tabla `ActivityInstance` que las otras actividades:

```sql
CREATE TABLE "Workflows"."ActivityInstance" (
    id serial4 PRIMARY KEY,
    activity int4 NOT NULL,              -- Vincula a definición de Actividad
    assignee text NULL,                  -- Asignado actual (nombre de usuario)
    "entityId" int4 NULL,                -- Registro de entidad en el que se trabaja
    "processInstance" int4 NOT NULL,     -- Vincula a ProcessInstance
    status "ActivityStateEnum" DEFAULT 'PENDING',
    "logActivity" text NULL,             -- Log de ejecución de actividad
    "result" text NULL                   -- Resultado de actividad
);
```

### Fase 2: Evaluación continua del workflow

#### 2.1 Motor de evaluación periódica de actividades
El motor de workflow evalúa continuamente todos los procesos activos:

```java
// PeriodicTasksVerticle se ejecuta cada 1 segundo
vertx.setPeriodic(1000, id -> {
    if (enabled) {
        InstancesModule.getInstancesService().runInstances();
    }
});
```

**Lógica de evaluación:**
1. **Obtener instancias de proceso activas** con estado `IN_PROGRESS`
2. **Para cada proceso**: Cargar todas las instancias de actividad
3. **Evaluar estado actual**: Determinar qué actividades pueden progresar
4. **Ejecutar transiciones**: Mover workflow a siguientes pasos cuando se cumplen condiciones

#### 2.2 Transiciones de estado de actividad
El sistema gestiona diferentes tipos de actividad:

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

### Fase 3: ejecución manual de actividades de tareas humanas

#### 3.1 Descubrimiento de tareas humanas (Task tray / Bandeja de tareas)
Los usuarios acceden a las actividades de tareas humanas pendientes a través de la interfaz de Bandeja de Tareas:

**Estructura de componentes frontend:**
- **MyTasksList** (`workflows/my-tasks-list/MyTasksList.tsx`): Interfaz principal de la Bandeja de tareas
- **MyTasksList** (`workflows/my-tasks-list/hooks/useMyTasksList.ts`): Hook con la funcionalidad de la pantalla
- **fetchUserTask && fetchUserGroupsTask** (`workflows/queries/`): Servicios para recuperar datos de la API GraphQL

**Consultas GraphQL para recuperación de tareas humanas:**
```graphql
# Actividades de tareas humanas sin asignar disponibles para grupos del usuario
query Workflows_ActivityInstanceList {
    result: Workflows_ActivityInstanceList(
        where: { 
            AND: [
                { status: { IN: [IN_PROGRESS] } },
                { assignee: { IS_NULL: true } },
                { activityDetail: { type: { EQ: "HUMAN_TASK" } } },
                { OR: [
                    { activityDetail: { userAssigned: { CONTAINS: [$username] } } },
                    { activityDetail: { groupAssigned: { CONTAINS: [$userGroups] } } }
                ]}
            ]
        }
    ) {
        id
        activity
        assignee
        entityId
        processInstance
        status
        activityDetail {
            name
            description
            entity
            entityMode
            type
            userAssigned
            groupAssigned
        }
        processInstanceDetail {
            businessKey
            process
        }
    }
}
```

#### 3.2 Asignación de actividad de tarea humana
**Antes de trabajar en una actividad de tarea humana**, los usuarios deben asignársela a sí mismos:

```typescript
// Llamada de servicio de asignación de actividad
const assignUserToActivity = async (activityId: number, username: string) => {
    await updateActivityInstanceAssignee(activityId, username);
    // El estado de la actividad permanece IN_PROGRESS pero ahora tiene asignado
};
```

**Estados de asignación:**
- **Sin asignar**: `assignee = null`, disponible para miembros del grupo
- **Asignado**: `assignee = username`, exclusivo para usuario asignado

#### 3.3 Apertura de formulario con contexto de workflow
Cuando el usuario hace clic en "Abrir Formulario" para una actividad de tarea humana, el sistema determina la URL correcta:

```typescript
const openForm = (activityInstance: ActivityInstance) => {
    const { activityDetail, processInstance, entityId } = activityInstance;
    
    let formUrl: string;
    
    if (activityDetail.entityMode === "NEW") {
        // Crear nueva entidad
        formUrl = `/admin/${activityDetail.entity}/new?processInstance=${processInstance}`;
    } else {
        // Editar entidad existente
        formUrl = `/admin/${activityDetail.entity}/${entityId}/edit?processInstance=${processInstance}`;
    }
    
    // Abrir en nueva ventana con contexto de workflow
    window.open(formUrl, '_blank');
};
```

**Parámetros de URL:**
- **`processInstance`**: Vincula formulario al contexto de workflow
- **`okUrl`**: URL de retorno opcional después de completar
- **Ruta de entidad**: Determina qué formulario cargar

### Fase 4: procesamiento de datos y creación de variables

#### 4.1 Detección de contexto del formulario
El componente EntityView detecta el contexto del workflow:

```javascript
// EntityView.js - Inicialización del formulario
save() {
    const okUrl = getParameter("okUrl", document.location.search);
    const processInstance = getParameter("processInstance", document.location.search);
    
    // La presencia de processInstance indica contexto de workflow
    if (processInstance != null) {
        // El formulario es parte del workflow - manejando actividad de tarea humana
    }
}
```

#### 4.2 Persistencia de datos de entidad
El formulario de entidades maneja tanto escenarios de crear como actualizar (create / update).


#### 4.3 Creación de variable de workflow
Después de guardar exitosamente la entidad, el sistema crea variables de workflow:

```javascript
invokeFallbackUrl(processInstance, entityName, entityId) {
    if (processInstance != null) {
        const query = 'mutation Create($name: String! $type: Workflows_VariableTypeEnumEnumType! ' +
            '$valueNumber: Int $processInstance: Int) { ' +
            'result: Workflows_VariableCreate(entity: { ' +
            'name: $name type: $type valueNumber: $valueNumber processInstance: $processInstance ' +
            '}) { id } }';

        const variables = {
            name: entityName,                    // ej: "Models.VacationRequest"
            description: entityName + '.' + entityId,  // ej: "Models.VacationRequest.123"
            type: "INTEGER",
            valueNumber: parseInt(entityId),     // El ID de la entidad creada/actualizada
            processInstance: parseInt(processInstance),
        };
        
        // Ejecutar mutación GraphQL para crear variable de workflow
        fetch('/graphql', {
            method: 'POST',
            body: JSON.stringify({query, variables})
        });
    }
}
```

**Propósito de la variable de workflow:**
- **Vincula entidad al proceso**: Crea relación entre datos de negocio y workflow
- **Dispara evaluación**: El motor de workflow detecta nueva variable y re-evalúa actividades
- **Habilita finalización**: Proporciona datos necesarios para condiciones de finalización de actividad de tarea humana

### Fase 5: manejo de resultados y finalización

#### 5.1 Lógica de finalización de actividad de tarea humana
El motor de workflow evalúa automáticamente cuándo las actividades de tareas humanas pueden completarse:

```java
private Future<Boolean> evaluateHumanTaskCompletion(
    ActivityInstanceDto activityInstanceDto,
    List<TransitionsDto> transitions,
    List<ActivityInstanceDto> activitiesInstance
) {
    return getProcessInstanceVariables(activityInstanceDto)
        .compose(mapVariables -> {
            
            if ("NEW".equalsIgnoreCase(activityInstanceDto.getActivityDetail().getEntityMode())) {
                // Modo NEW: Completar cuando existe variable de workflow para entidad
                return Future.succeededFuture(
                    mapVariables.containsKey(activityInstanceDto.getActivityDetail().getEntity())
                );
            }
            
            // Modo EXISTING: Evaluar condiciones de transición
            List<ActivityInstanceDto> completedHumanTaskActivities = getCompletedHumanActivities(activitiesInstance);
            completedHumanTaskActivities.add(activityInstanceDto);
            
            return updateMapVariablesWithEntities(mapVariables, completedHumanTaskActivities)
                .map(updatedMap -> {
                    var nextSteps = transitions.stream()
                        .filter(t -> t.getSourceActivity().equals(activityInstanceDto.getActivityDetail().getId()))
                        .toList();
                    
                    var nextActivities = activitiesInstance.stream()
                        .filter(activity -> nextSteps.stream().anyMatch(transition ->
                            transition.getTargetActivity().equals(activity.getActivityDetail().getId())
                                && evaluateCondition(transition.getConditionFunction(), updatedMap)))
                        .toList();
                    
                    return !nextActivities.isEmpty();
                });
        });
}
```

**Modos de finalización:**

**Modo NEW Entity:**
- **Disparador**: Variable de workflow creada con nombre de entidad como clave
- **Condición**: `mapVariables.containsKey(entityName)`
- **Ejemplo**: Variable "Models.VacationRequest" existe → actividad de tarea humana puede completarse

**Modo EXISTING Entity:**

El modo EXISTING es más complejo que el modo NEW, utilizando un sistema de evaluación basado en condiciones de las transiciones que parten del nodo a evaluar.

**¿Qué entidad debe actualizar el usuario?**

En el modo EXISTING, el usuario actualiza un **registro de entidad preexistente** que ya está vinculado a la instancia de actividad (mediante el campo `entityId` de la actividad):

> NOTA: Esta funcionalidad no está soportada 100% a día de hoy, ya que a través del formulario de creación de actividad manual nsolo se puede proveer una función de "selección", cuya finalidad en principio sería seleccionar el ID del registro que tiene que actualizar el usuario, pero esta función de selección no se está utilizando.

2. **Generación de URL del formulario**: Cuando el usuario hace clic en "Abrir formulario" para una actividad en modo EXISTING, el sistema genera una URL que abre el formulario de edición para el registro específico:

```typescript
if (activityDetail.entityMode === "EXISTING") {
    // Editar entidad existente
    formUrl = `/admin/${activityDetail.entity}/${entityId}/edit?processInstance=${processInstance}`;
}
```

**Ejemplo**: Si tienes un workflow de aprobación de solicitud de vacaciones, la actividad podría estar vinculada a un registro existente `Models.VacationRequest` con ID 123. El usuario actualizaría campos como `status`, `approverComments`, `approvedDays`, etc. en ese registro específico.

**¿Cómo sabe el backend cuando la tarea manual está completa?**

Esta es la parte más sofisticada de la lógica del modo EXISTING. El backend utiliza un **sistema de evaluación basado en condiciones** en lugar de simple detección de variables:

1. **Proceso de evaluación periódica**: El motor de workflow se ejecuta cada 1 segundo y evalúa todas las tareas humanas IN_PROGRESS.

2. **Carga de datos de entidad y evaluación**: Para el modo EXISTING, el sistema:
   - **Carga datos actuales de entidad**: Obtiene todos los atributos del registro de entidad vinculado usando `updateMapVariablesWithEntities()`
   - **Crea claves de variables**: Los atributos de entidad se vuelven disponibles en el contexto de evaluación con dos patrones de nomenclatura:
     - `{activityUniqueId}_{attributeName}` (ej: `approve_vacation_status`)
     - `{entityName}_{attributeName}` (ej: `Models_VacationRequest_status`)

3. **Evaluación de condiciones de transición**: Verifica si alguna transición saliente de la actividad actual tiene sus condiciones satisfechas:

```java
// Desde EvaluateActivitiesUseCase.java
return updateMapVariablesWithEntities(mapVariables, completedHumanTasks)
    .map(updatedMap -> {
        var nextSteps = transitions.stream()
            .filter(t -> t.getSourceActivity().equals(activityInstanceDto.getActivityDetail().getId()))
            .toList();

        var nextActivities = activitiesInstance.stream()
            .filter(activity -> nextSteps.stream().anyMatch(transition ->
                transition.getTargetActivity().equals(activity.getActivityDetail().getId())
                    && evaluateCondition(transition.getConditionFunction(), updatedMap)))
            .toList();

        return !nextActivities.isEmpty(); // La tarea se completa si las siguientes actividades están disponibles
    });
```

4. **Evaluación de condiciones**: Las condiciones se evalúan usando expresiones JEXL que pueden referenciar los datos de entidad:

```java
private Boolean evaluateCondition(String condition, Map<String, Object> variables) {
    try {
        Object result = ees.evaluate(condition, variables); // Evaluación JEXL
        return result instanceof Boolean && (Boolean) result;
    } catch (Exception e) {
        log.error("Error evaluating condition '{}': {}", condition, e.getMessage());
        return false;
    }
}
```

**Punto clave**: La tarea se completa cuando **al menos una condición de transición saliente se vuelve verdadera** basada en el estado actual de los datos de entidad.

**Ejemplos concretos:**

*Ejemplo 1: Workflow de aprobación de solicitud de vacaciones*
- Entidad: `Models.VacationRequest` (ID: 123)
- Actividad: "Aprobación del manager" (modo EXISTING)
- Condición de transición: `Models_VacationRequest_status == "APPROVED" || Models_VacationRequest_status == "REJECTED"`

Flujo:
1. Usuario abre el formulario: `/admin/Models.VacationRequest/123/edit?processInstance=456`
2. Usuario actualiza el registro: `status` = "APPROVED", `approverComments` = "¡Disfruta tus vacaciones!"
3. Usuario guarda el formulario (no se crea variable de workflow en modo EXISTING)
4. Evaluación del backend (cada 1 segundo): Carga datos de entidad, evalúa condición → **verdadera** → La tarea se completa automáticamente

*Ejemplo 2: Workflow de revisión de documentos*
- Entidad: `Models.Document` (ID: 789)
- Actividad: "Revisión legal" (modo EXISTING)
- Condición de transición: `Models_Document_legalStatus == "APPROVED" && Models_Document_reviewerSignature != null`

**Diferencias clave del modo NEW:**

| Aspecto | Modo NEW | Modo EXISTING |
|---------|----------|---------------|
| **Entidad** | Crea nuevo registro | Actualiza registro existente |
| **Disparador de finalización** | Creación de variable de workflow | Evaluación de condición de transición |
| **URL del formulario** | `/admin/{entity}/new?processInstance={id}` | `/admin/{entity}/{entityId}/edit?processInstance={id}` |
| **Creación de variable** | Siempre crea variable de workflow | No se crea variable de workflow |
| **Evaluación de condición** | Simple: la variable existe | Compleja: evaluación de expresión JEXL |

**Notas importantes sobre el modo EXISTING:**

1. **No se crean variables de workflow**: A diferencia del modo NEW, el modo EXISTING no crea variables de workflow cuando se guarda el formulario.
2. **Evaluación en tiempo real**: El backend monitorea continuamente el estado de la entidad y completa automáticamente la tarea cuando se cumplen las condiciones.
3. **Condiciones flexibles**: Puedes crear condiciones complejas como:
   - `Models_Invoice_status == "APPROVED" && Models_Invoice_amount < 10000`
   - `Models_Contract_signedBy != null && Models_Contract_legalReview == true`
4. **Múltiples rutas de transición**: Una sola tarea humana puede tener múltiples transiciones salientes con diferentes condiciones, permitiendo lógica de ramificación compleja basada en decisiones del usuario.

- **Disparador**: Condiciones de transición se satisfacen basadas en datos de entidad actualizados
- **Condición**: Al menos una transición saliente evalúa como verdadera usando expresiones JEXL
- **Ejemplo**: `Models_VacationRequest_status == "APPROVED"` → actividad de tarea humana de aprobación se completa automáticamente

#### 5.2 Ejecución de finalización de actividad
Cuando se cumplen las condiciones de finalización:

```java
public void completeActivityInstance(Integer activityInstanceId) {
    try {
        if (updateActivityInstance(activityInstanceId, "DONE")) {
            createActivityStatus(activityInstanceId, ActivityInstanceStatusEnum.DONE.toString());
        }
    } catch (Exception e) {
        log.error("Error completing activity: {}", e.getMessage());
    }
}
```

**Acciones de finalización:**
1. **Actualizar estado**: `IN_PROGRESS` → `DONE`
2. **Crear registro de estado**: Registrar timestamp de finalización
3. **Disparar evaluación**: Siguientes actividades se vuelven disponibles
4. **Continuar workflow**: Proceso avanza a pasos subsiguientes

### Fase 6: disponibilidad de resultados para actividades posteriores

#### 6.1 Evaluación de transición
Después de la finalización de actividad, el sistema evalúa las transiciones del workflow:

```java
// Obtener transiciones para actividad completada
return transitionsApi.fetchTransitions(activityIds)
    .compose(transitions -> getNextActivities(completedActivity, transitions, activitiesInstance))
    .compose(nextActivities -> processNextActivities(nextActivities, transitions, activitiesInstance));
```

#### 6.2 Activación de siguiente actividad
El sistema activa actividades subsiguientes:

```java
private Future<Boolean> processNextActivities(List<ActivityInstanceDto> nextActivities,
    List<TransitionsDto> transitions, List<ActivityInstanceDto> activitiesInstance) {
    
    var nextPending = nextActivities.stream()
        .filter(a -> ActivityInstanceStatusEnum.PENDING.toString().equalsIgnoreCase(a.getStatus()))
        .toList();
    
    var nextInProgress = nextActivities.stream()
        .filter(a -> ActivityInstanceStatusEnum.IN_PROGRESS.toString().equalsIgnoreCase(a.getStatus()))
        .toList();
    
    // Iniciar actividades pendientes
    nextPending.forEach(this::startActivity);
    
    // Evaluar actividades en progreso para finalización
    nextInProgress.forEach(activity -> evaluateActivityIfComplete(activity, transitions, activitiesInstance));
    
    return Future.succeededFuture(true);
}
```

### Fase 7: manejo de errores

#### 7.1 Procesamiento de errores en actividades manuales
Cuando una actividad manual falla o encuentra problemas:

```java
private void handleHumanTaskError(ActivityInstanceDto activityInstanceDto, String errorMessage) {
    log.error("Error in human task activity: {}", errorMessage);
    
    // Marcar actividad como error
    instancesApi.errorActivityInstance(activityInstanceDto.getId());
    
    // Actualizar log con información del error
    instancesApi.updateLogActivityInstance(
        activityInstanceDto.getId(),
        errorMessage,
        "Error in Human Task: " + errorMessage
    );
}
```

#### 7.2 Estados de error
Las actividades manuales pueden tener los siguientes estados de error:
- **ERROR**: La actividad falló durante la ejecución o procesamiento
- **Log actualizado**: Se registra el mensaje de error para diagnóstico
- **Workflow bloqueado**: El proceso no puede continuar hasta resolver el error

## Resumen del flujo completo

### Secuencia del flujo de datos para actividades manuales
1. **Inicio de proceso** → Todas las instancias de actividad creadas con estado `PENDING`
2. **Descubrimiento de actividades** → Los usuarios ven actividades de tareas humanas disponibles en bandeja de actividades vía GraphQL
3. **Asignación de actividad** → Usuario se asigna la actividad de tarea humana a sí mismo (`assignee` actualizado)
4. **Apertura de formulario** → Sistema genera URL con parámetro `processInstance`
5. **Finalización de formulario** → Usuario guarda datos de entidad vía mutación GraphQL
6. **Creación de variable** → Frontend crea variable de workflow vinculando entidad al proceso
7. **Evaluación automática** → Backend detecta variable y evalúa condiciones de finalización
8. **Finalización de actividad** → Estado cambia a `DONE`, historial de estado actualizado
9. **Progresión del workflow** → Siguientes actividades activadas basadas en transiciones
10. **Continuación del proceso** → Ciclo se repite para actividades de tareas humanas subsiguientes

### Transiciones de estado de actividad manual
```
PENDING → IN_PROGRESS → DONE
   ↓           ↓          ↓
Creado    Asignado   Completado
          e Iniciado  y Evaluado
```

### Diferencias clave con otras actividades

| Aspecto | Actividades Manuales | Actividades de Función | Actividades de IA |
|---------|---------------------|----------------------|-------------------|
| **Inicio** | Requiere asignación y acción del usuario | Automático cuando se cumplen condiciones | Automático cuando se cumplen condiciones |
| **Ejecución** | Formulario completado por usuario | Función ejecutada por sistema | Agente de IA invocado vía función |
| **Parámetros** | Datos de entidad del formulario | Variables evaluadas del contexto del workflow | Agent name + prompt enriquecido |
| **Resultado** | Variable de workflow de entidad creada/actualizada | Variables de workflow con resultado de función | Solo log de actividad |
| **Tiempo** | Indeterminado (depende del usuario) | Inmediato (limitado por tiempo de ejecución) | Depende del agente de IA externo |
| **Asignación** | Requiere `assignee` | No requiere asignación | No requiere asignación |

### Puntos clave de integración

#### Configuración de actividad manual
- **Activity Name**: Nombre identificativo de la actividad
- **Entity**: Nombre de la entidad que se creará o editará
- **Entity Mode**: `NEW` (crear) o `EXISTING` (editar)
- **Users/Groups Assigned**: Usuarios y grupos autorizados para la actividad

#### Evaluación de finalización
- **Referencias de entidad**: `Application.Entity.Field`
- **Variables de proceso**: Nombre directo de variable
- **Resultados de actividades**: `uniqueId.field`
- **Condiciones de transición**: Pueden referenciar datos de entidad usando expresiones JEXL

#### Resultados y continuación
- **Variables de workflow**: Se crean automáticamente al guardar entidades
- **Condiciones de transición**: Pueden referenciar variables usando expresiones JEXL
- **Disponibilidad**: Variables disponibles tan pronto como se guarda la entidad

#### Frontend (workflows) ↔ Backend
- **API GraphQL**: Consulta de actividades y operaciones de creación/actualización de entidades
- **Parámetros de URL**: Paso de contexto de workflow (`processInstance`) 
- **Persistencia de variables del workflow en BD**: Mecanismo de vinculación entidad-proceso

#### Frontend (admin tool) ↔ Backend 
- **Detección de contexto**: Presencia de parámetro `processInstance`
- **Creación de variables**: Función `invokeFallbackUrl` después de creación/actualización de entidades
- **Evaluación automática**: Evaluación periódica del motor de workflows

#### Usuario ↔ Sistema
- **Bandeja de actividades**: Interfaz visual para gestión de actividades de tareas humanas
- **Formularios de entidades**: Creación/actualización de un registro de una entidad vincualado a un workflow
