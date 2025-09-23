## Cómo las entidades referencian otras entidades

### 1. Definición de referencia
Una referencia de entidad se define dentro de una entidad usando la palabra clave `reference`:

```airflows
entity Demo.Product {
    // ... otros atributos ...
    
    reference category {
        attribute category
        basicFilter
        cascadeDelete
        label
        list
        listIsVisible
        referencedKey: Demo.Category.Category_pkey
        visible
    }
}
```

### 2. Componentes clave de una referencia

**Componentes requeridos:**
- `attribute` - El atributo local que almacena el valor de clave foránea
- `referencedKey` - Apunta a la clave específica en la entidad referenciada (formato: `EntityName.KeyName`)

**Configuración opcional:**
- `basicFilter` - Habilita filtrado básico en esta referencia
- `cascadeDelete` - Elimina registros relacionados cuando se elimina el padre
- `cascadeSetNull` - Establece clave foránea a nulo cuando se elimina el padre
- `label` - Muestra etiqueta en UI
- `list` - Muestra en vistas de lista
- `listIsVisible` - Hace la lista visible por defecto
- `listIsFilteredWhenEmpty` - Aplica filtros cuando está vacío
- `visible` - Hace la referencia visible en formularios
- `linkDisabled` - Deshabilita navegación a entidad referenciada
- `additionalAttributes` - Atributos adicionales para mostrar
- `additionalFilter` - Expresiones de filtro personalizadas
- `group` - Agrupa la referencia en UI
- `tab` - Coloca referencia en pestaña específica
- `order` - Orden de visualización
- `sm`, `xs` - Tamaño responsivo

### 3. Atributos de referencia
La referencia debe especificar qué atributos locales participan en la relación:

```airflows
reference category {
    attribute category  // Este es el atributo local que contiene la clave foránea
    referencedKey: Demo.Category.Category_pkey
    // ... otras opciones
}
```

### 4. Formato de clave referenciada
El `referencedKey` usa el formato: `EntityName.KeyName`

Del ejemplo de demo:
- `Demo.Category.Category_pkey` - Referencia la clave primaria de la entidad Category
- La entidad referenciada debe tener una clave definida con ese nombre

### 5. Ejemplo completo del demo

```airflows
entity Demo.Product {
    attribute category {
        label es_ES: "Categoría"
        label en_US: "Category"
        type: INTEGER  // Esto contiene el valor de clave foránea
    }
    
    // ... otros atributos ...
    
    reference category {
        attribute category                    // Apunta al atributo local de arriba
        basicFilter                          // Habilita filtrado
        cascadeDelete                        // Elimina productos cuando se elimina categoría
        label                                // Muestra etiqueta en UI
        list                                 // Muestra en vistas de lista
        listIsVisible                        // La lista es visible por defecto
        referencedKey: Demo.Category.Category_pkey  // Referencia la clave primaria de Category
        visible                              // Muestra en formularios
    }
}
```

### 6. Cómo funciona en la práctica

1. **Almacenamiento de clave foránea**: El atributo local (ej., `category`) almacena el ID de la entidad referenciada
2. **Integración de UI**: La referencia proporciona capacidades de navegación y visualización
3. **Integridad de datos**: Las opciones de cascada mantienen la integridad referencial
4. **Filtrado**: Las referencias pueden usarse para filtrado y búsqueda
5. **Navegación**: Los usuarios pueden navegar de una entidad a sus entidades referenciadas

## Ejemplo de uso (del demo)

El demo muestra una aplicación de tienda de ropa con:
- **Entidad Category**: Categorías de productos con imágenes y búsqueda
- **Entidad Product**: Productos con precios, códigos de barras, clasificación de género
- **Enum Gender**: Masculino/Femenino con colores e iconos
- **Rol Customer**: Con permisos específicos para ver y gestionar datos
- **Funciones**: Actualizaciones de búsqueda de texto y cálculos de ranking
- **Enlaces externos**: Integración con sistemas externos


Cuando el usuario crea una referencias desde la UI , en primer lugar crea un campo de tipo entero para guardar la futura referencia a otra entidad:

EntityAttribute 
id|container|name   |type   |isArray|
--+---------+-------+-------+-------+
65|        3|id     |SERIAL |false  |
68|        3|id_user|INTEGER|false  | 

Luego, asocia este campo de tipo entero a la clave primaria de otra entidad mediante el diagrama E/R. En ese momento, el core de Airflows crea la clave fóranea sobre el campo que había creado previamente el usuario e inserta un registro en EntityRefernce:

EntityReference
id|container|name   |referencedKey|isList|
--+---------+-------+-------------+------+
 1|        3|id_user|            4|true  |
