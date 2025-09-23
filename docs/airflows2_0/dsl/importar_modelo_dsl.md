# importar un modelo escrito en Airflows DSL

## ¿por qué hace falta un paso intermedio?

* El backend solo entiende dos formatos de importación:
  1. un libro Excel (.xls)
  2. un ZIP cuya raíz contiene **model.json** (+ ficheros auxiliares)
* Los ficheros **.airflows** son texto DSL; antes de importarlos hay que **compilarlos** y generar ese ZIP.

## paso A – compilar el paquete DSL

1. Instala la herramienta de línea de mando o el plugin de VS Code:
   ```bash
   npm i -g @airflows/dsl-tools   # CLI
   ```
2. Desde la carpeta que contiene tus *.airflows* ejecuta:
   ```bash
   airflows-dsl build .
   ```
   Se crea `build/model-package.zip`, que incluye **model.json**.

## paso B – importar en la instancia local

### a) usando la interfaz web

1. Inicia sesión como administrador.
2. Menú → Admin → Export / Import → pestaña “Import”.
3. Arrastra `model-package.zip` y espera la confirmación.

### b) usando cURL / script

```bash
curl -X POST http://localhost:8080/model-import \
     -H "Authorization: Bearer <TOKEN>" \
     -H "Content-Type: application/zip" \
     --data-binary @build/model-package.zip
```

## qué ocurre después

* El código servidor (`ModelImportUseCase`) procesa el ZIP, crea las tablas (por ejemplo `aia_agent`, `aia_tool`, …) y carga los datos.
* Si el usuario dispone del privilegio **MENU** para esas entidades, el ítem “AI Agents” aparece automáticamente en el menú React.

## problemas frecuentes

| síntoma | causa | solución |
|---------|-------|----------|
| errores de FK al importar | existe un modelo previo incompatible | limpia la BD (`./gradlew :com.niledb.platform:deleteDatabase`) o usa una base nueva |
| “model.json not found” | el ZIP no fue generado con la herramienta DSL | vuelve a ejecutar `airflows-dsl build` |
| el menú no se muestra | falta privilegio MENU o entidad no presente | revisa privilegios y confirma que las tablas `aia_*` existen |

Con esto, el modelo de agentes de IA definido en DSL queda operativo en tu base de datos local.
