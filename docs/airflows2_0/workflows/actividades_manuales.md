---
sidebar_position: 3
---

# Actividades manuales en workflows

## Introducción

Las actividades manuales (tipo `HUMAN_TASK`) son puntos de interacción humana dentro de un flujo de trabajo automatizado. Estas actividades permiten que los usuarios creen, modifiquen o revisen entidades del sistema como parte del proceso de negocio, integrando perfectamente la intervención humana con la automatización del workflow.

## Arquitectura y componentes

### 1. Tipos de actividad en el sistema

El sistema de workflows soporta varios tipos de actividades definidos en `ActivityTypeEnum.java`:

```java
public enum ActivityTypeEnum {
    HUMAN_TASK,      // Tarea manual que requiere intervención humana
    AI_AGENT_TASK,   // Tarea ejecutada por agente IA
    FUNCTION,        // Función automática
    AND,             // Compuerta lógica AND
    OR,              // Compuerta lógica OR
    XOR,             // Compuerta lógica XOR
    START,           // Inicio del proceso
    END              // Fin del proceso
}
```

### 2. Estructura de una actividad manual

Una actividad de tipo `HUMAN_TASK` contiene los siguientes campos principales (`Activity.java`):

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `id` | Long | Identificador único de la actividad |
| `name` | String | Nombre descriptivo de la actividad |
| `description` | String | Descripción detallada |
| `entity` | String | Nombre de la entidad a gestionar |
| `entityMode` | String | Modo de operación: "NEW" o existente |
| `entitySelectionFunction` | String | Expresión para seleccionar la entidad |
| `uniqueId` | String | Identificador único para variables |
| `groupAssigned` | String[] | Grupos que pueden completar la tarea |
| `userAssigned` | String[] | Usuarios específicos asignados |

### 3. Estados del ciclo de vida

Las instancias de actividades manuales transitan por los siguientes estados:

1. **PENDING**: Estado inicial al crear la instancia
2. **IN_PROGRESS**: Cuando la tarea está activa y esperando acción del usuario
3. **DONE**: Tarea completada exitosamente
4. **ERROR**: Error durante la ejecución

## Flujo de ejecución detallado

### Fase 1: Creación de la instancia

Cuando se inicia un proceso de workflow (`StartProcessUseCase.java`):

```java
masterProcess.getActivities().forEach(activity -> {
    entityUtils.getEntityIdFromVars(activity.getEntity(), processInstancesDto.getId())
        .onSuccess(entityId -> {
            var activityInstance = ActivityInstanceDto.builder()
                .activity(activity.getId())
                .entityId(entityId.orElse(null))  // null para entidades nuevas
                .processInstance(masterProcess.getId())
                .status(ActivityInstanceStatusEnum.PENDING.toString())
                .build();
            
            instancesApi.createActivityInstance(activityInstance, processInstancesDto.getId());
        });
});
```

**Puntos clave:**
- Todas las actividades se crean con estado `PENDING`
- El `entityId` es `null` para modo NEW, o contiene el ID existente para modo EXISTING
- Se registra el estado inicial en la tabla de auditoría

### Fase 2: Activación de la tarea

En `EvaluateActivitiesUseCase.java`, cuando una tarea manual debe iniciarse:

```java
private void startActivity(ActivityInstanceDto activityInstanceDto) {
    instancesApi.openActivityInstance(activityInstanceDto.getId());
    
    switch (activityInstanceDto.getActivityDetail().getType()) {
        case "HUMAN_TASK":
            // NO se ejecuta ningún servicio automático
            // La tarea queda esperando intervención humana
            break;
        case "AI_AGENT_TASK":
            InstancesModule.getAiExecutorService().execute(...);
            break;
        case "FUNCTION":
            InstancesModule.getFunctionsExecutorService().execute(...);
            break;
    }
}
```

**Diferencia clave**: A diferencia de las tareas automáticas, las `HUMAN_TASK` no invocan ningún executor service y simplemente esperan la acción del usuario.

### Fase 3: Interacción del usuario

#### 3.1 Navegación desde la bandeja de tareas

El usuario accede a sus tareas pendientes a través de una interfaz que:
1. Lista las actividades con estado `IN_PROGRESS` y tipo `HUMAN_TASK`
2. Filtra por usuario o grupos asignados
3. Proporciona un enlace con el contexto del workflow:

```
/admin/{NombreEntidad}/new?processInstance={idInstanciaProceso}&okUrl={urlRetorno}
```

#### 3.2 Creación/edición de la entidad

Cuando el usuario guarda la entidad, el frontend (`useOnSaveEntity.tsx`):

```typescript
const processInstance = searchParams.get('processInstance')
const okUrl = searchParams.get('okUrl')

// Después de guardar exitosamente la entidad
const id = getCreatedEntityId(entityState, saveEntityResponse)
if (id && processInstance) {
    // Crear variable de proceso para señalizar la creación
    invokeFallbackUrl(processInstance, entity, id)
}
```

### Fase 4: Registro de la variable del proceso

#### Función `invokeFallbackUrl`

Esta función crítica (`invokeFallbackUrl.ts`) crea una variable de proceso que vincula la entidad con el workflow:

```typescript
export const invokeFallbackUrl = async (
  processInstance: string,
  entityName: string,
  entityId: string,
): Promise<void> => {
  const query = `
    mutation Create (
      $name: String!
      $description: String
      $type: Workflows_VariableTypeEnumEnumType!
      $valueNumber: Int
      $processInstance: Int
    ) {
      result: Workflows_VariableCreate(
        entity: {
          name: $name
          description: $description
          type: $type
          valueNumber: $valueNumber
          processInstance: $processInstance
        }
      ) { id }
    }`

  const variables = {
    name: entityName,                     // Nombre del tipo de entidad
    description: `${entityName}.${entityId}`,
    type: 'INTEGER',
    valueNumber: parseInt(entityId),      // ID de la entidad creada
    processInstance: parseInt(processInstance),
  }
  
  await graphqlQueryRaw(query, variables)
}
```

**Propósito**: Esta variable sirve como señal de que la entidad ha sido creada/actualizada, permitiendo al motor de workflow evaluar si la tarea puede completarse.

### Fase 5: Evaluación de completitud

El método `evaluateHumanTaskCompletion()` verifica si una tarea manual puede marcarse como completada:

#### Modo NEW

```java
if ("NEW".equalsIgnoreCase(activityInstanceDto.getActivityDetail().getEntityMode())) {
    return Future.succeededFuture(
        mapVariables.containsKey(
            activityInstanceDto.getActivityDetail().getEntity()
        )
    );
}
```

**Lógica**: Verifica si existe una variable con el nombre de la entidad. Si existe, significa que el usuario creó la entidad y la tarea puede completarse.

#### Modo EXISTING

Para entidades existentes, el proceso es más complejo:

```java
// 1. Obtener entidades humanas completadas
List<ActivityInstanceDto> completedHumanTasks = getCompletedHumanActivities(activitiesInstance);
completedHumanTasks.add(activityInstanceDto);

// 2. Actualizar variables con datos de entidades
return updateMapVariablesWithEntities(mapVariables, completedHumanTasks)
    .map(updatedMap -> {
        // 3. Evaluar condiciones de transición
        var nextSteps = transitions.stream()
            .filter(t -> t.getSourceActivity()
                .equals(activityInstanceDto.getActivityDetail().getId()))
            .toList();
        
        // 4. Verificar si hay actividades siguientes válidas
        var nextActivities = activitiesInstance.stream()
            .filter(activity -> nextSteps.stream().anyMatch(transition ->
                transition.getTargetActivity().equals(activity.getActivityDetail().getId())
                && evaluateCondition(transition.getConditionFunction(), updatedMap)))
            .toList();
        
        return !nextActivities.isEmpty();
    });
```

#### Enriquecimiento de variables

El método `updateMapVariablesWithEntities` obtiene los valores de los atributos de la entidad y los agrega a las variables del proceso:

```java
public Future<Map<String, Object>> updateMapVariablesWithEntities(
    Map<String, Object> mapVariables,
    List<ActivityInstanceDto> completedHumanTasks) {
    
    completedHumanTasks.stream().forEach(activity ->
        entityApi.fetchEntityDetail(activity.getActivityDetail().getEntity())
            .compose(fields -> entityApi.fetchAllAttributeValues(
                activity.getActivityDetail().getEntity(),
                activity.getEntityId(),
                fieldNames
            ))
            .onSuccess(attributes -> {
                for (AttributeDto attribute : attributes) {
                    // Patrón 1: {uniqueId de actividad}_{nombre campo}
                    String key1 = activity.getActivityDetail().getUniqueId().replace(".", "_") 
                                + "_" + attribute.getName();
                    // Patrón 2: {nombre entidad}_{nombre campo}
                    String key2 = activity.getActivityDetail().getEntity().replace(".", "_") 
                                + "_" + attribute.getName();
                    
                    mapVariables.put(key1, attribute.getValue());
                    mapVariables.put(key2, attribute.getValue());
                }
            })
    );
}
```

### Fase 6: Completitud de la tarea

Cuando se determina que la tarea puede completarse:

```java
private void completeActivity(ActivityInstanceDto activityInstanceDto) {
    instancesApi.completeActivityInstance(activityInstanceDto.getId());
    // Actualiza el estado a "DONE"
    // Registra en la tabla de auditoría
    // Permite que el workflow continúe
}
```

## Modos de operación de entidades

### Modo NEW

**Características:**
- La actividad espera la creación de una nueva entidad
- El `entityId` inicial es `null`
- Se completa cuando existe una variable con el nombre de la entidad
- Usado para procesos que requieren registro de nuevos datos

**Flujo:**
1. Usuario navega desde bandeja de tareas con `processInstance` en URL
2. Crea nueva entidad en el formulario
3. Al guardar, `invokeFallbackUrl` crea variable del proceso
4. El motor detecta la variable y marca la tarea como completable

### Modo EXISTING

**Características:**
- La actividad trabaja con una entidad existente
- El `entityId` se obtiene de las variables del proceso al inicio
- Puede actualizar o simplemente revisar la entidad
- Se completa basándose en condiciones de transición

**Flujo:**
1. El sistema identifica la entidad desde las variables del proceso
2. Usuario navega a la entidad existente con contexto del workflow
3. Realiza modificaciones necesarias
4. Los valores de la entidad se usan para evaluar transiciones

## Integración frontend-backend

### Parámetros de navegación

| Parámetro | Propósito | Origen |
|-----------|-----------|--------|
| `processInstance` | ID de la instancia del proceso | Bandeja de tareas |
| `okUrl` | URL de retorno después de guardar | Configuración del workflow |
| `entityId` | ID de entidad existente (modo EXISTING) | Variables del proceso |

### Discriminación de contexto

El sistema distingue entre operaciones normales y operaciones de workflow mediante:

```typescript
// Operación normal (sin workflow)
if (!processInstance) {
    // Guardar entidad normalmente
    // No crear variables de proceso
}

// Operación de workflow
if (processInstance) {
    // Guardar entidad
    // Invocar invokeFallbackUrl
    // Crear variable de proceso
}
```

## Patrones de uso de variables

Las variables del proceso se utilizan para:

1. **Identificación de entidades**: Variable con nombre de entidad y valor numérico (ID)
2. **Valores de campos**: Variables con patrón `{entidad}_{campo}` o `{uniqueId}_{campo}`
3. **Evaluación de condiciones**: Usadas en expresiones JEXL para transiciones

### Normalización de nombres

Debido a las limitaciones de JEXL con la notación de puntos:
- Los puntos (`.`) se reemplazan por guiones bajos (`_`)
- Ejemplo: `Models.Customer` → `Models_Customer`

## Consideraciones de seguridad

1. **Asignación de tareas**: Solo usuarios/grupos asignados pueden ver y completar tareas
2. **Aislamiento de proceso**: Las variables están limitadas a la instancia del proceso
3. **Validación de contexto**: El `processInstance` debe ser válido para crear variables

## Manejo de errores

### Errores comunes y soluciones

| Error | Causa | Solución |
|-------|-------|----------|
| Variable no creada | Navegación directa sin `processInstance` | Acceder desde bandeja de tareas |
| Tarea no se completa | Variable de entidad no existe | Verificar `invokeFallbackUrl` |
| EntityId null en modo EXISTING | Variable de proceso faltante | Crear entidad en paso previo |

## Monitoreo y debugging

### Puntos de registro clave

1. **Creación de actividad**: `StartProcessUseCase` - log de actividades creadas
2. **Cambios de estado**: `InstancesApi` - transiciones de estado
3. **Evaluación de completitud**: `EvaluateActivitiesUseCase` - condiciones evaluadas
4. **Creación de variables**: `invokeFallbackUrl` - variables registradas

### Consultas útiles para debugging

```sql
-- Ver tareas manuales pendientes
SELECT ai.*, a.entity, a.entity_mode 
FROM activity_instance ai
JOIN activity a ON ai.activity = a.id
WHERE ai.status = 'IN_PROGRESS' 
  AND a.type = 'HUMAN_TASK';

-- Verificar variables de proceso
SELECT * FROM variable 
WHERE process_instance = ? 
  AND name = 'NombreEntidad';

-- Historial de estados de una actividad
SELECT * FROM activity_state 
WHERE activity_instance = ?
ORDER BY timestamp;
```

## Mejores prácticas

1. **Diseño de workflows**:
   - Usar modo NEW para creación de registros maestros
   - Usar modo EXISTING para actualizaciones y revisiones
   - Definir claramente los grupos/usuarios asignados

2. **Nomenclatura**:
   - Usar `uniqueId` descriptivos para las actividades
   - Mantener nombres de entidad consistentes
   - Documentar el propósito de cada tarea manual

3. **Gestión de errores**:
   - Implementar timeouts para tareas críticas
   - Proporcionar rutas de escalamiento
   - Registrar todas las interacciones importantes

4. **Experiencia de usuario**:
   - Proporcionar contexto claro en la bandeja de tareas
   - Incluir instrucciones específicas en la descripción
   - Usar `okUrl` para navegación fluida

## Conclusión

Las actividades manuales son el puente entre la automatización de procesos y la necesaria intervención humana. Su diseño permite una integración transparente donde:

- Los usuarios trabajan con las interfaces familiares del admin tool
- El contexto del workflow se mantiene mediante parámetros URL
- La creación de variables automatiza el seguimiento del progreso
- La evaluación de condiciones permite flujos complejos de decisión

Esta arquitectura proporciona flexibilidad para implementar procesos de negocio complejos mientras mantiene la simplicidad para los usuarios finales.
