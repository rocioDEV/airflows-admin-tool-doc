---
sidebar_position: 6
---

# queryParam `processInstance`


1. **Uso en la interfaz de usuario (EntityView.js)**:
   - El parámetro se obtiene de la URL usando: `getParameter("processInstance", document.location.search)`
   - Se utiliza principalmente en el contexto de flujos de trabajo (workflows)
   - Se convierte a entero usando `parseInt(processInstance)` cuando se usa en operaciones

2. **Integración con GraphQL**:
   - Se utiliza en una mutación GraphQL para crear variables de workflow:
   ```graphql
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
     ) {
       id
     }
   }
   ```

3. **Estructura de Base de Datos**:
   - Existe una tabla `Workflows.ProcessInstance` que contiene los siguientes campos:
     - `businessKey`
     - `creationDate`
     - `process`
     - `status`

4. **Relaciones en la Base de Datos**:
   - Se relaciona con la tabla `Workflows.ProcessState` donde `processInstance` es una columna que referencia a `ProcessInstance`
   - También se relaciona con la tabla `Workflows.Variable` donde `processInstance` es una columna que almacena la referencia a la instancia del proceso

5. **Flujo de Trabajo**:
   - Cuando se crea una nueva instancia de proceso:
     1. Se inserta un registro en `Workflows.ProcessInstance`
     2. Se crea un estado inicial en `Workflows.ProcessState`
     3. Se pueden crear variables asociadas en `Workflows.Variable`

6. **Comportamiento en la UI**:
   - Se utiliza en conjunto con un parámetro `okUrl` para el manejo de redirecciones después de operaciones
   - Existe un método `invokeFallbackUrl` que maneja la lógica posterior a las operaciones cuando hay un `processInstance` presente

7. **Formato del Business Key**:
   - Se genera un business key con el formato: `'Holidays_Requests_' || NEW.id || '_' || TO_CHAR(NOW(), 'YYYYMMDD')`

Este parámetro parece ser una parte fundamental del sistema de workflows, actuando como un identificador que conecta diferentes aspectos del proceso de workflow, incluyendo sus estados y variables asociadas.