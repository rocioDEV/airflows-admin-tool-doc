---
sidebar_position: 2
---

# Documentación técnica del tipo de atributo "selector externo"

## Descripción general

El `externalSelector` es un tipo de atributo especial que permite mostrar una interfaz externa través de un pop-up para que el usuario seleccione el valor del atributo. Proporciona una forma de **seleccionar valores desde fuentes externas** mientras mantiene el contexto del formulario actual.

## Propiedades de configuración

(en el orden en el que aparecen el el formulario de creación/edición de campos, sección **External selector**)
![image](https://hackmd.io/_uploads/ByXvHI6mlx.png)

### 1. externalSelectorUrl

- **Tipo**: String
- **Descripción**: La URL de la interfaz de selector externo
- **Uso**: Es la URL que se renderizará en el pop up con la interfaz de selección externa
- **Características**:
  - Soporta sustitución de variables usando la sintaxis `${variable.name}`
  - Puede incluir token de acceso si `addAccessToken` está habilitado

### 2. externalSelectorTarget

- **Tipo**: String
- **Descripción**: Especifica la ventana objetivo para el selector externo
- **Valor por defecto**: "\_blank"
- **Valores válidos**:
  - "\_self": Se abre en la misma ventana
  - "\_blank": Se abre en una nueva ventana/pestaña
  - "\_parent": Se abre en el frame padre
  - "\_top": Se abre en el cuerpo completo de la ventana
  - Nombre personalizado: Se abre en una ventana con nombre

### 3. externalSelectorWindowFeatures

- **Tipo**: String
- **Descripción**: Lista de características de la ventana separadas por comas
- **Ejemplo**: "popup,width=400,height=300"

### 4. addAccessToken

- **Tipo**: Boolean
- **Descripción**: Controla si se debe añadir el token de acceso a la URL del selector externo
- **Valor por Defecto**: false
- **Uso**: Cuando es true, añade el parámetro "access_token" a la URL para autenticación

### 5. externalSelectorEditableValue

- **Tipo**: Boolean
- **Descripción**: Determina si el valor seleccionado puede ser editado manualmente
- **Valor por Defecto**: false
- **Uso**: Cuando es true, permite a los usuarios modificar el valor seleccionado directamente en el campo de entrada (en Airflows)

## Componentes de la interfaz

El selector externo se renderiza como una combinación de:

1. Un campo de entrada de texto
2. Un botón de búsqueda (con icono de búsqueda)
3. Un botón de limpieza (con icono de limpieza)

## Funcionalidad / Casos de uso

### Selección de un valor

1. El usuario hace clic en el botón de búsqueda
2. El sistema abre la URL del selector externo en la ventana objetivo especificada
3. El usuario realiza una selección en la interfaz externa
4. La selección se comunica de vuelta a la aplicación principal mediante mensajes entre ventanas

### Manejo de valores

- El valor seleccionado puede ser:
  - Valor directo (cuando `externalSelectorEditableValue` es true)
  - Valor de referencia (cuando se usa `referenceAttributeName`)

### Funcionalidad de limpieza

- El botón de limpieza permite a los usuarios restablecer el valor seleccionado
- Al limpiar:
  - Se vacía el campo de entrada
  - El valor de referencia (si existe) se establece como null

## Integración

### Comunicación entre ventanas

- El campo añade un listener a nivel de window para escuchar mensajes enviados desde la ventana que renderiza el selector externo
- Recibe el valor seleccionado en la ventana externa a través del manejador de eventos `handleMessage`

```tsx
// se añade el listener a nivel de window
window.addEventListener("message", handleMessage);
// cuando la pantalla de selección externa envía el mensaje,
// se captura el evento y se actualiza el valor en el formulario
const handleMessage = (event: MessageEvent) => {
  field.onChange(event.data.value);
};
```

- Desde la ventana de selección externa el único requisito es implementar una función que pase el valor seleccionado por el usuairo a la ventana que la invocó a través de `postMessage`

```tsx
window.opener?.postMessage(
  {
    value: SELECTED_VALUE,
    entity: params.entity,
    attribute: params.attribute,
  },
  "*"
);
```

### Manejo de parámetros url

La URL del selector externo puede incluir los siguientes parámetros:

- language: Idioma actual de la interfaz
- entity: Nombre de la entidad actual
- entityId: ID de la entidad actual
- attribute: Nombre del atributo que se está seleccionando
- access_token: (Opcional) Cuando addAccessToken está habilitado

## Ejemplo de valores de configuración

```javascript
{
  "name": "campoExterno",
  "externalSelectorUrl": "https://selector-externo.ejemplo.com",
  "externalSelectorTarget": "_blank",
  "externalSelectorWindowFeatures": "popup,width=600,height=400",
  "externalSelectorEditableValue": false,
  "addAccessToken": true
}
```

## Limitaciones y notas

2. Depende del soporte de la API de mensajes entre ventanas
3. Puede ser bloqueado por bloqueadores de ventanas emergentes si no está configurado correctamente
4. Requiere configuración CORS adecuada para URLs externas
