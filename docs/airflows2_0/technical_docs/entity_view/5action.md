---
sidebar_position: 5
---

# Tipos de atributo: acción

## Descripción general

El campo `action` es una propiedad booleana de un campo de entidad que determina si el atributo está asociado con un comportamiento accionable. Cuando se establece en `true`, el atributo se trata como una acción que puede activar funcionalidades específicas, como abrir una URL, ejecutar un comando u otras operaciones. Es decir, permite renderizar un botón en lugar de un campo para introducir valores, para facilitar que el usuario pueda introducir una acción personalizada dentro del formulario o el listado.

---

## Detalles del campo

### **Nombre del campo:** `action`

### **Tipo:** Booleano

### **Propósito:**

El campo `action` se utiliza para definir si un atributo representa un elemento accionable dentro de la aplicación. Se utiliza principalmente en componentes de renderizado y para manejar interacciones del usuario.

---

## Referencias de código

### **Renderización de botones de acción**

El campo `action` se utiliza en la clase `EntityList` para renderizar botones para atributos accionables:

```javascript
// filepath: EntityList.js
render() {
    /*...código existente...*/
    entityLocalModel.attributes[attribute.name].action
        && item[attribute.name] != null
        && item[attribute.name] != ""
        && (
            <Button
                data-qa={attribute.name + "-button"}
                variant="contained"
                onClick={event => this.handleActionClick(event, item, attribute)}
            >
                {t('e.' + this.props.entity + '.a.' + attribute.name)}
            </Button>
        )
    /*...código existente...*/
}
```

### **Manejo de clics de acción**

El método `handleActionClick` en la clase `EntityView` procesa el comportamiento asociado con atributos accionables:

```javascript
// filepath: EntityView.js
handleActionClick(event, item, attribute) {
    event.preventDefault();
    event.stopPropagation();
    if (item[attribute.name] != null && item[attribute.name] != "") {
        let url = item[attribute.name];
        url += (url.indexOf("?") != (-1) ? "&" : "?") + "access_token=" + this.props.context.accessToken;
        fetch(url, {
            method: "GET",
            redirect: "manual",
        })
        .then(response => {
            let location = response.headers.get("Location");
            if (location != null && location != "") {
                document.location = location;
            }
        });
    }
}
```

### **Visibilidad en la interfaz de usuario**

El campo `action` también se verifica para determinar si el campo debe ser visible en ciertos contextos de la interfaz de usuario:

```javascript
// filepath: EntityList.js
render() {
    /*...código existente...*/
    attribute.array
        && !entityLocalModel.attributes[attribute.name].action
        && attribute.enumType != null
        && item[attribute.name] != null
        && (
            <div>{item[attribute.name].join(", ")}</div>
        )
    /*...código existente...*/
}
```

---

## Campos relacionados

- **`actionVisibleInForm`:** Determina si la acción es visible en formularios.
- **`actionVisibleInToolbar`:** Especifica la visibilidad en componentes de barra de herramientas.

---

## Ejemplo

Un atributo con el campo `action` establecido en `true` podría representar una URL que abre una nueva página al hacer clic:

```json
{
  "name": "openLink",
  "type": "TEXT",
  "action": true,
  "actionVisibleInForm": true,
  "actionVisibleInToolbar": false
}
```

---
