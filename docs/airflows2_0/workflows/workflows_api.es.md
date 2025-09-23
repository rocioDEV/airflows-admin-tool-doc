# Workflows API

## visión de arquitectura
El Workflows API sigue un patrón de **gestión manual de dependencias** con singletons estáticos y una arquitectura por capas. Está diseñado para ejecutar workflows tipo BPMN con soporte para varios tipos de actividad, incluidas tareas de agente IA.

## componentes core

### 1. gestión de módulos (`InstancesModule.java`)
**Propósito**: contenedor central de dependencias y factoría de servicios.
**Patrón**: inicialización estática con cableado manual.

```java
public class InstancesModule {
  private static final InstancesService instancesService;
  private static final InstancesApi instancesApi;
  private static final VariableApi variableApi;
  private static final AiExecutorService aiExecutorService;
  
  static {
    // Creación y cableado manual
    variableRepository = new VariableRepository();
    variableApi = new VariableApi(variableRepository, modelMapper);
    aiExecutorService = new AiExecutorService(instancesApi, functionsApi, variableApi);
  }
}
```

### 2. capa API

#### `InstancesApi.java`
- Gestión del ciclo de vida de instancias de proceso y actividad
- Métodos clave: `fetchProcessInstance`, `createActivityInstance`, `updateStatusProcessInstance`, `completeActivityInstance`, `errorActivityInstance`

#### `VariableApi.java`
- Gestión del ciclo de vida de variables de workflow
- Métodos: `fetchVariables`, `createProcessInstanceVariables`

#### `FunctionsApi.java`
- Ejecución de funciones y evaluación de variables
- Métodos: `invokeFunction`, `evaluateVariables`, `parseFunctionVariables`

### 3. capa de casos de uso

#### `AiExecutorService.java`
- Orquestación de tareas IA
- Enriquecimiento de prompts con contexto del workflow
- Integración de variables
- Invocación de funciones IA

#### `EvaluateActivitiesUseCase.java`
- Ruteo por tipo de actividad
```java
switch (activityInstanceDto.getActivityDetail().getType()) {
  case "AI_AGENT_TASK":
    InstancesModule.getAiExecutorService().execute(taskId, activityInstanceDto);
    break;
  // ...
}
```

#### otros
- `StartProcessUseCase`, `EvaluateEndProcessUseCase` para iniciar y finalizar procesos

### 4. DTOs principales

`ActivityInstanceDto`, `ActivityDto`, `VariableDto` (ver estructuras en el original; incluyen detalles de actividad, variables multi-tipo y referencias a instancia de proceso/actividad).

### 5. capa repositorio

- `VariableRepository`, `ProcessInstanceRepository`, `ActivityInstanceRepository` para acceso a BD.

### 6. entidades

- `Variable`, `ProcessInstance`, `ActivityInstance` como representaciones de BD.

## flujos de datos

### flujo de tarea IA
1. Detección de `AI_AGENT_TASK`
2. Enrutado a `AiExecutorService.execute()`
3. Enriquecimiento del prompt con contexto y variables
4. Invocación de funciones vía `FunctionsApi`
5. Resolución de variables y ejecución
6. Actualización de estado y logging

### flujo de resolución de variables
1. `VariableApi.fetchVariables()` recupera variables
2. Se anexan al prompt y a variables de función
3. El agente IA recibe el contexto completo

## configuración y dependencias
- Base de datos PostgreSQL
- Jackson para JSON
- Vert.x (Futures) para async
- SLF4J/Logback para logging

## manejo de errores
- Fallos de variables: fallback con logging
- Errores de función: marca actividad como ERROR
- IDs inválidos, problemas de BD: logging exhaustivo

## rendimiento
- Operaciones async con Futures
- Singletons para reutilización
- Optimización de acceso a BD

## seguridad
- Variables acotadas a la instancia de proceso
- Validación/sanitización de parámetros de función

## extensibilidad
- Nuevos tipos de actividad: ampliar switch y módulo
- Funciones personalizadas: repositorio/registro y configuración
- Nuevos tipos de variable: extender DTO, esquema y lógica

## observabilidad
- Logging de depuración, métricas y seguimiento de estado

## consideraciones futuras
- Inyección de dependencias (p.ej., Spring)
- Externalizar configuración
- Métricas de rendimiento
- Capa de caché (p.ej., Redis)
- Documentación API (OpenAPI/Swagger)
