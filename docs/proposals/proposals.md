# 🌠 Propuestas de mejora

- [🌠 Propuestas de mejora](#-propuestas-de-mejora)
    - [PARA IMPLEMENTAR](#para-implementar)
    - [PARA DISCUTIR](#para-discutir)

### PARA IMPLEMENTAR
-  ~~Propuesta nuevo menú con dos niveles~~:
<iframe src="https://codesandbox.io/embed/2qjrtn?view=preview&hidenavigation=1"
     style={{ width:"100%", height: "500px", border:0, borderRadius: "4px", overflow:"hidden" }}
     title="stage3-recursive-menu-item (forked)"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>

- ~~Debería aparercer tooltip en las opciones de menú con título muy largo~~
- ~~La **configuración específica de cada tipo de campo** se debería mostrar u ocultar dependiendo del tipo de campo seleccionado~~
- ~~Si se ha discontinuado la parte de **"Informes"**, tiene sentido quitarla?~~
- ~~El selector de color se puede esconder hasta que se haga click (mostrando un cuadrado pintado y el hexadecimal en modo visualización)~~
- ~~Navegación por breadcrumbs~~
- ~~Cuando una entidad no tiene seleccionado campo para identificar el formu, el breadcrumb muestra el nombre vacío. Mostrar  "Sin etiqueta" con un tootip que indique que hay que marcar un campo como identificador del formulario para que aparezca algún valor~~

- Prerellenar el formulario de grupos/pestañas y mantener los datos al volver al formulario de creación /edición de campo
- No se pueden añadir campos a una entidad en modo editar
- El menú lateral debe desparecer cuando hay una sola opción
- El menú lateral debe aparecer plegado por defecto cuando hay pocas opciones y desplegado en caso contrario (>5)
- En configuración de un campo, "**Types of accepted files**" debería haber alguna pista o validación del formato que esperamos (example: .zip)
- Al filtrar los resultados de una tabla, debería corregirse el **orden de los campos de búsqueda** (no se respeta el del formulario)
- Cuando una **configuración de campo no está soportada** (texto, marco casilla de icono pero no añado lista de valores), presentar informe de errores en el formulario en vez de pintar el literal `NOT_SUPPORTED`
- En modo anonimo, debería aparecer el literal "Salir" en vez de "Cerrar sesión"
- El **ancho del menú** podría ser configurable y debería aparercer tooltip en las opciones de menú con título muy largo 
- Marcar visualmente los elementos del menú nuevos (entidades que se acaban de crear)

_____________
### PARA DISCUTIR
- Una vez que se establece una relación, habría que **limpiar las opciones que se pueden aplicar** ("aparece en el formulario" hace que el campo aparezca dos veces)
- Debería **unificarse** en código Formularios - Forms //  Campos - fields? *(Domain Driven Design*)
- Al crear aplicación, tiene sentido que **"Nombre aplicación"** sea un campo de texto libre?
- Cambiar validación de campos para que sea un poco más obvia
