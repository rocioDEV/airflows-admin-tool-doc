# Problemas detectados al importar un modelo y cómo solucionarlos

## 1. FileNotFoundException durante la importación

**Síntoma**  
`java.io.FileNotFoundException: /data/model/model/applications/aia/functions/updateTextSearchModel.pgsql`

**Causa**  
El importador descomprime el ZIP en la ruta absoluta `/data/model`. Luego concatena esta ruta con el valor de `path` que figura en `model.json`.  
Si el directorio `/data/model` no existe en la máquina local, los archivos no se escriben y al intentar leerlos se lanza la excepción.

**Solución en entorno local**
```bash
sudo mkdir -p /data/model
sudo chown $USER /data/model
```
Vuelve a importar el ZIP después de crear la carpeta.


## 2. Error DNS al acceder a `/admin/aia.Assistant/external`

**Síntoma**  
El navegador muestra:  
`xxxxx.flows.ninja’s server IP address could not be found.`

**Causa**  
El registro correspondiente en la tabla `Models.ExternalEntity` contiene la URL por defecto `https://xxxxx.flows.ninja/...`.  
El frontend redirige automáticamente a esa URL al abrir la vista `/external`.

**Soluciones**

1. **Modificar la URL desde la interfaz:**
   - Navegar a **Models → External entity**.
   - Editar la fila `aia.Assistant`.
   - Cambiar el campo **url** por la dirección local, por ejemplo `http://localhost:8080/assistant`.
   - Guardar.

2. **Ajustar la URL antes de importar:**
   - Descomprimir el ZIP.
   - Editar `model.json` (o el fichero correspondiente) y sustituir el valor de `url`.
   - Volver a comprimir e importar.

3. **Actualizar directamente en PostgreSQL:**
   ```sql
   UPDATE "Models"."ExternalEntity"
   SET    url = 'http://localhost:8080/assistant',
          addAccessToken = false
   WHERE  schema = 'aia'
     AND  name   = 'Assistant';
   ```

Después de aplicar cualquiera de estas correcciones el modelo se importa sin errores y los enlaces externos funcionan de forma local.
