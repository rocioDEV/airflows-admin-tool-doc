- Propuesta nuevo menú con dos niveles:
<iframe src="https://codesandbox.io/embed/2qjrtn?view=preview&hidenavigation=1"
     style="width:100%; height: 500px; border:0; border-radius: 4px; overflow:hidden;"
     title="stage3-recursive-menu-item (forked)"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>

- La **configuración específica de cada tipo de campo** se debería mostrar u ocultar dependiendo del tipo de campo seleccionado
- Al filtrar los resultados de una tabla, debería corregirse el **orden de los campos de búsqueda** (no se respeta el del formulario)
- Cuando una **configuración de campo no está soportada** (texto, marco casilla de icono pero no añado lista de valores), presentar informe de errores en el formulario en vez de pintar el literal `NOT_SUPPORTED`
- En modo anonimo, debería aparecer el literal "Salir" en vez de "Cerrar sesión"
- Al crear aplicación, tiene sentido que **"Nombre aplicación"** sea un campo de texto libre?
- El **ancho del menú** podría ser configurable y debería aparercer tooltip en las opciones de menú con título muy largo 

_____________
- Si se ha discontinuado la parte de **"Informes"**, tiene sentido quitarla?
- Al pasar un campo de texto a campo enumerado, el mensaje de error debería ser más descriptivo (sale **error SQL**)
- Una vez que se establece una relación, habría que **limpiar las opciones que se pueden aplicar** ("aparece en el formulario" hace que el campo aparezca dos veces)
- En configuración de un campo, "**Types of accepted files**" debería haber alguna pista o validación del formato que esperamos (example: .zip)
- Debería **unificarse** en código Formularios - Forms //  Campos - fields? *(Domain Driven Design*)
