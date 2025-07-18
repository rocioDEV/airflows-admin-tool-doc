---
sidebar_position: 2
---

# Guardado de datos de una entidad (`useOnSaveEntity`)

- [Guardado de datos de una entidad (`useOnSaveEntity`)](#guardado-de-datos-de-una-entidad-useonsaveentity)
  - [**Propósito**](#propósito)
  - [**Parámetros**](#parámetros)
  - [**Mapeos**](#mapeos)
    - [**1. Filtrado de Atributos**](#1-filtrado-de-atributos)
    - [**2. Construcción de Consultas GraphQL**](#2-construcción-de-consultas-graphql)
    - [**3. Manejo de Errores**](#3-manejo-de-errores)
    - [**4. Navegación y Actualización de Estado**](#4-navegación-y-actualización-de-estado)
  - [**Llamadas a la API**](#llamadas-a-la-api)
    - [**1. Endpoint GraphQL**](#1-endpoint-graphql)
    - [**2. Invocación de URL de Respaldo**](#2-invocación-de-url-de-respaldo)
  - [**Flujo Lógico**](#flujo-lógico)
  - [**Navegaciones**](#navegaciones)
    - [**Caso: `entity === 'Models.Personalization'`**](#caso-entity--modelspersonalization)
    - [**Caso: `entity.startsWith('Models.')`**](#caso-entitystartswithmodels)
    - [**Caso: Otros (entidades que no comienzan con `Models.`)**](#caso-otros-entidades-que-no-comienzan-con-models)
    - [**Caso: Creación de una nueva entidad (sin `entityId`)**](#caso-creación-de-una-nueva-entidad-sin-entityid)
  - [**Diagrama**](#diagrama)


## **Propósito**

El hook `useOnSaveEntity` se encarga de gestionar la lógica para guardar o actualizar una entidad en la aplicación. Procesa los valores del formulario, valida los atributos, construye consultas GraphQL y realiza llamadas a la API para guardar o actualizar la entidad en el backend. También maneja la navegación y actualizaciones de estado según el resultado de las operaciones.

---

## **Parámetros**
El hook recibe los siguientes parámetros:

| Parámetro          | Tipo                                   | Descripción                                                                 |
|--------------------|----------------------------------------|-----------------------------------------------------------------------------|
| `entity`           | `string`                              | Nombre de la entidad que se está guardando o actualizando.                  |
| `entityId`         | `string`                              | Identificador único de la entidad. Si está definido, se actualiza la entidad; de lo contrario, se crea una nueva. |
| `entityState`      | `EntityState`                         | Estado actual de la entidad, incluyendo atributos y claves principales.     |
| `mode`             | `'view' | 'edit'`                     | Modo de operación: `'view'` o `'edit'`.                                     |
| `inputRefs`        | `Record<string, React.RefObject>`      | Referencias a los campos del formulario.                                    |
| `baseUrl`          | `string`                              | URL base para las llamadas a la API.                                        |
| `accessToken`      | `string`                              | Token de acceso para autenticación.                                         |
| `model`            | `ParsedModel`                         | Modelo parseado que contiene metadatos sobre la entidad.                    |
| `setEntityState`   | `Dispatch<SetStateAction<EntityState>>`| Función para actualizar el estado de la entidad.                            |

---

## **Mapeos**

El hook realiza los siguientes mapeos y transformaciones sobre los parámetros:

### **1. Filtrado de Atributos**

- Los atributos se filtran utilizando la función `filterAttributes` para determinar cuáles deben incluirse en la consulta GraphQL.
- **Condiciones**:
  - Excluir atributos con `incomingReferenceAttributeName`.
  - Incluir atributos según su tipo (e.g., `TEXT`, `POINT`, `DOCUMENT`, `BPMN_EDITOR`).
  - Verificar si el atributo tiene un valor en `formValues`.

### **2. Construcción de Consultas GraphQL**

- **Para actualizaciones**:
  - Se llama a la función `updateEntity` con los atributos de la entidad, valores del formulario y otros parámetros.
- **Para creación**:
  - Se construye dinámicamente una consulta de mutación GraphQL utilizando los atributos filtrados y sus tipos.
  - Las variables se rellenan con valores de `formValues` utilizando la función `getVariableValue`.

### **3. Manejo de Errores**

- Los errores de la respuesta de la API se mapean a mensajes amigables para el usuario utilizando la función `getErrorMessage`.

### **4. Navegación y Actualización de Estado**

- Si la operación es exitosa, el hook actualiza el estado de la entidad (`setEntityState`) o navega a una URL específica utilizando `navigate` o `history.replace`.

---

## **Llamadas a la API**

El hook realiza las siguientes llamadas a la API:

### **1. Endpoint GraphQL**

- **URL**: `${baseUrl}/graphql`
- **Método**: `POST`
- **Payload**:
  - Para actualizaciones: El payload es construido por la función `updateEntity`.
  - Para creación: Una consulta de mutación GraphQL construida dinámicamente y variables.

### **2. Invocación de URL de Respaldo**

- **Función**: `invokeFallbackUrl`
- **Propósito**: Si la entidad no pertenece al espacio de nombres `Models.`, esta función construye y envía una mutación GraphQL para crear una entidad `Workflows_Variable`.
- **URL**: `${baseUrl}/graphql`
- **Método**: `POST`
- **Payload**: Una consulta de mutación con variables predefinidas.

---

## **Flujo Lógico**

1. **Validación**:

   - Verifica si `entity`, `entityId` y `accessToken` están definidos.
   - Si no, termina la ejecución.

2. **Flujo de Actualización**:

   - Si `entityId` está definido:
     - Llama a `updateEntity` con los parámetros necesarios.
     - Maneja la respuesta (errores o éxito).
     - Navega o actualiza el estado según la respuesta.

3. **Flujo de Creación**:

   - Si `entityId` no está definido:
     - Filtra los atributos utilizando `filterAttributes`.
     - Construye una consulta de mutación GraphQL para la creación.
     - Rellena las variables con valores de `formValues`.
     - Realiza una solicitud `POST` al endpoint GraphQL.
     - Maneja la respuesta (errores o éxito).
     - Navega o actualiza el estado según la respuesta.

4. **Lógica de Respaldo**:
   - Si la entidad no pertenece al espacio de nombres `Models.`, invoca la función `invokeFallbackUrl` para manejar la operación.

---

## **Navegaciones**

El hook realiza las siguientes navegaciones en cada caso:

### **Caso: `entity === 'Models.Personalization'`**

- **Navegación**:
  - Si `okUrl` está definido: `document.location.replace(okUrl)`
  - Si no: `history.replace('/admin/' + entity + '/' + entityId + '/view')`
- **Descripción**:
  - Redirige al usuario a la URL especificada o a la vista de la entidad personalizada.

---

### **Caso: `entity.startsWith('Models.')`**

- **Navegación**:
  - Si `okUrl` está definido: `document.location.replace(okUrl)`
  - Si no: `history.replace('/admin/' + entity + '/' + entityId + '/view')`
- **Descripción**:
  - Redirige al usuario a la URL especificada o a la vista de la entidad.

---

### **Caso: Otros (entidades que no comienzan con `Models.`)**

- **Navegación**:
  - Si `okUrl` está definido: `document.location.replace(okUrl)`
  - Si no: `history.replace('/admin/' + entity + '/' + (entityState.keyAttribute?.name && json.data.result[entityState.keyAttribute?.name]) + '/view')`
- **Descripción**:
  - Redirige al usuario a la URL especificada o a la vista de la entidad utilizando su clave principal.

---

### **Caso: Creación de una nueva entidad (sin `entityId`)**

- **Navegación**:
  - Si `okUrl` está definido: `document.location.replace(okUrl)`
  - Si no: `navigate('/admin/' + entity + '/' + entityId + '/view')`
- **Descripción**:
  - Redirige al usuario a la URL especificada o a la vista de la nueva entidad creada.

---

## **Diagrama**

Aquí tienes un diagrama de alto nivel del flujo lógico:

```
Entradas:
  - entity, entityId, entityState, mode, inputRefs, baseUrl, accessToken, model, setEntityState

Lógica:
  1. Validar entradas
  2. Si entityId existe:
     - Llamar a updateEntity
     - Manejar respuesta (errores/éxito)
     - Navegar o actualizar estado
  3. Si entityId no existe:
     - Filtrar atributos
     - Construir consulta de mutación GraphQL
     - Rellenar variables
     - Realizar llamada a /graphql
     - Manejar respuesta (errores/éxito)
     - Navegar o actualizar estado
  4. Si la entidad no pertenece a Models.:
     - Llamar a invokeFallbackUrl
     - Realizar llamada a /graphql
     - Manejar respuesta
```

---
