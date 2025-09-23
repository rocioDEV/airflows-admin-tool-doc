# Inicialización de base de datos y deltas

### Visión general

Esta guía explica cómo se crea el esquema de base de datos de Airflows, cómo se rastrean las versiones del esquema y cuándo se aplican los deltas SQL numerados.

- **Esquema base**: se crea mediante `com.niledb.platform/misc/model/create.sql`.
- **Versión de esquema**: se registra en `"Models"."Version"`.
- **Deltas**: viven en `com.niledb.platform/misc/model/sql/deltas/` y se aplican en orden numérico ascendente al arrancar el backend.

### Flujo de inicialización

Existen dos caminos que inicializan el esquema de la base de datos por primera vez:

- **primer arranque con docker compose**: Postgres ejecuta `create.sql` automáticamente porque está montado en `docker-entrypoint-initdb.d`.

  Ejemplo de la configuración de compose:

```yaml
# compose.yaml
services:
  db:
    image: hub.airflows.com:5443/airflows/postgres:15
    volumes:
      - ./com.niledb.platform/misc/model/create.sql:/docker-entrypoint-initdb.d/00_create.sql
```

- **respaldo del backend (sin compose/BD nueva)**: Si el backend no puede conectarse a la base de datos especificada (catálogo inválido), crea la base de datos, ejecuta `misc/model/create.sql` y luego reinicia el contenedor `db` para reactivar `pg_cron`.

### Qué hace create.sql

El script base realiza estas tareas:

- Garantiza que exista la extensión `pg_cron`.
- Crea el esquema `"Models"` y establece el `search_path`.
- Define tipos, tablas, funciones, triggers y primitivas de seguridad utilizadas por Airflows.
- Crea la tabla de versión `"Models"."Version"` (el valor inicial lo establece el primer delta).

Fragmentos clave:

```sql
CREATE EXTENSION IF NOT EXISTS pg_cron;

CREATE SCHEMA IF NOT EXISTS "Models";
SET search_path TO "Models";

CREATE TABLE IF NOT EXISTS "Models"."Version" (
  "version" int NOT NULL
);
```

### Versionado de esquema y archivos delta

- La versión actual del esquema se lee de `"Models"."Version"` al arrancar el backend. Si la tabla o la fila no existen, la versión por defecto es 0 y se inicializará mediante deltas.
- Los archivos delta son scripts SQL planos, nombrados con un prefijo entero (p. ej., `1.sql`, `2.sql`, ..., `144.sql`).
- Al arrancar, el backend:
  - Enumera los archivos en `misc/model/sql/deltas/`
  - Los ordena numéricamente por el prefijo del nombre
  - Para cada archivo con `fileNumber > currentVersion`:
    - Lee el archivo y elimina las líneas que comienzan con comentarios `--`
    - Ejecuta el SQL resultante
    - Actualiza `"Models"."Version"` al número del archivo ejecutado

El primer delta (`1.sql`) inicializa la tabla de versión:

```sql
CREATE TABLE IF NOT EXISTS "Models"."Version" (
  "version" int NOT NULL
);
DELETE FROM "Models"."Version";
INSERT INTO "Models"."Version" VALUES (1);
```

### ¿Cuándo se aplican los deltas?

Los deltas se aplican durante el arranque del backend después de establecer una conexión correcta a la base de datos y leer la versión actual. Esto ocurre:

- Justo después de que el esquema base esté en su lugar (vía primer arranque con compose o respaldo del backend).
- En cada arranque posterior del backend, cualquier delta nuevo cuyo número sea mayor que `"Models"."Version"` se aplica.

### Cómo añadir un nuevo delta

1. Determina el siguiente entero después del delta existente de mayor número en `misc/model/sql/deltas/`.
2. Crea un archivo nuevo llamado `<N>.sql` con tus cambios SQL.
3. Usa `--` para comentarios. Las líneas que empiezan por `--` son ignoradas por el aplicador; los comentarios de bloque `/* ... */` no se eliminan.
4. No actualices manualmente `"Models"."Version"` dentro del archivo. El backend lo actualiza automáticamente a `<N>` si el script se completa correctamente.

Ejemplo de nombre de archivo: `misc/model/sql/deltas/145.sql`.

### Comprobación y resolución de problemas

- **Comprobar la versión actual**:

```sql
SELECT "version" FROM "Models"."Version";
```

Usando psql localmente (compose expone la BD en localhost:5433):

```bash
psql -h localhost -p 5433 -U postgres -d niledb -c 'SELECT "version" FROM "Models"."Version";'
```

Usando docker compose para ejecutar psql dentro del contenedor de BD:

```bash
docker compose exec -T db psql -U postgres -d niledb -c 'SELECT "version" FROM "Models"."Version";'
```

- **Volver a ejecutar un delta**: El aplicador solo ejecuta archivos con números mayores que la versión actual. Para volver a ejecutar un delta, crea un nuevo delta con un número mayor que contenga los cambios correctivos.

- **Idempotencia**: Prefiere SQL idempotente (p. ej., `CREATE ... IF NOT EXISTS`, `ALTER ... IF EXISTS`) para hacer los deltas seguros entre entornos.

### Notas operativas y advertencias

- Los deltas se ejecutan como un lote concatenado por archivo sin envoltura transaccional explícita por parte del aplicador. Un fallo puede aplicar parcialmente un delta y detener actualizaciones posteriores. Diseña deltas resilientes y fáciles de recuperar.
- Usa comentarios de línea `--`. Los comentarios de bloque no se eliminan por el aplicador y se envían a la base de datos tal cual.
- Con docker compose, eliminar el volumen de la BD provoca que `create.sql` se ejecute de nuevo en el siguiente arranque, recreando las estructuras base antes de que se (re)apliquen los deltas.


