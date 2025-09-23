# Análisis del sistema de privilegios de entidades

## Resumen

Este análisis investigó el sistema actual de privilegios para entidades en la plataforma Airflows, buscando específicamente código que asigne automáticamente privilegios por defecto (SELECT y MENU) a nuevas entidades cuando son creadas por usuarios.

## Arquitectura del sistema actual

### Componentes clave

1. **Almacenamiento de entidades**: La tabla `Models.Entity` almacena las definiciones de entidades
2. **Almacenamiento de privilegios**: La tabla `Models.EntityPermission` almacena permisos a nivel de entidad con tipos como SELECT, MENU, EXPORT
3. **Trigger de creación de entidades**: La función `entity_trigger_after` maneja las tareas posteriores a la creación

### Ubicaciones de archivos

- **Esquema de base de datos**: `/home/rderandom/dev/airflows-uno/com.niledb.platform/misc/model/create.sql`
- **Asignación manual de privilegios**: `/home/rderandom/dev/airflows-uno/com.niledb.platform/src/main/java/tools/usecase/ModelImportUseCase.java` (líneas 2235-2280)
- **Lectura de privilegios**: `/home/rderandom/dev/airflows-uno/com.niledb.platform/src/main/java/graphql/resolvers/GetModelFetcher.java` (líneas 232-258)

## Proceso actual de creación de entidades

Cuando se crea una nueva entidad, la función `entity_trigger_after` (líneas 1270-1280 en `create.sql`) actualmente solo:

1. Crea un atributo `id` por defecto con tipo SERIAL
2. Crea una restricción de clave primaria

```sql
CREATE OR REPLACE FUNCTION "Models"."entity_trigger_after"() RETURNS trigger AS $$
DECLARE
    sql text;
begin
    sql := 'WITH attribute AS (INSERT INTO "Models"."EntityAttribute"(container, name, type, "isRequired") VALUES (' || new.id || ', ''id'', ''SERIAL'', true) RETURNING id), key AS (INSERT INTO "Models"."EntityKey"(container, schema, name, "isUnique", "isPrimaryKey") VALUES (' || new.id || ', ''' || new.schema || ''', ''' || new.name || '_pkey'', true, true) RETURNING id) INSERT INTO "Models"."EntityKeyAttribute" (container, attribute) SELECT key.id, attribute.id FROM key, attribute';
    EXECUTE sql;

    return new; 
end
$$ LANGUAGE plpgsql;
```

## Funcionalidad faltante

El sistema **carece de asignación automática de privilegios** cuando se crean nuevas entidades. El comportamiento esperado sería insertar automáticamente privilegios por defecto como:

```json
{
  "entities": {
    "geo.Pueblos": {
      "schema": "geo",
      "name": "Pueblos",
      "privileges": {
        "SELECT": "SELECT",
        "MENU": "MENU"
      }
    }
  }
}
```

## Patrones actuales de asignación de privilegios

### Asignación manual (ModelImportUseCase.java)
El sistema tiene código funcional para la asignación de privilegios durante las importaciones de modelos:

```java
PreparedStatement ps3 = connection.prepareStatement(
    "INSERT INTO \"Models\".\"EntityPermission\""
        + " (\"role\", \"entity\", \"types\") "
        + " VALUES (?, ?, ?::\"Models\".\"entityPermissionType\"[]) "
        + " ON CONFLICT (\"role\", \"entity\") "
        + " DO UPDATE SET \"types\" = EXCLUDED.\"types\" ");
```

### Lectura de privilegios (GetModelFetcher.java)
El sistema lee correctamente los privilegios MENU:

```java
// El privilegio MENU se toma de la tabla EntityPermission
if (username.equals("anonymous")) {
    sql = "SELECT schemaname, tablename from (SELECT e.schema AS schemaname, e.name AS tablename, unnest(types) AS permissiontype FROM \"Models\".\"Entity\" e, \"Models\".\"Role\" r, \"Models\".\"EntityPermission\" ep WHERE ep.role = r.id AND ep.entity = e.id AND r.name IN (?)) AS record WHERE record.permissiontype = 'MENU'";
}
```

## Solución recomendada

**Mejorar la función `entity_trigger_after`** en `/home/rderandom/dev/airflows-uno/com.niledb.platform/misc/model/create.sql` para insertar automáticamente registros EntityPermission por defecto para el rol anonymous con privilegios SELECT y MENU.

La implementación debería:

1. Obtener el ID del rol anonymous
2. Insertar registros EntityPermission con tipos SELECT y MENU
3. Seguir el mismo patrón usado en ModelImportUseCase.java

Esto aseguraría que cada nueva entidad reciba automáticamente los privilegios por defecto sin requerir intervención manual.

## Tablas de base de datos involucradas

- **`Models.Entity`**: Almacena definiciones de entidades
- **`Models.EntityPermission`**: Almacena asignaciones de privilegios (rol, entidad, array de tipos)
- **`Models.Role`**: Contiene roles incluyendo 'anonymous'

Los tipos de privilegios se almacenan como arrays de enum de PostgreSQL en la columna `types` de la tabla EntityPermission.
