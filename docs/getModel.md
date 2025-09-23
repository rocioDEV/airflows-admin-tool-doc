## Cómo se resuelven las consultas getModel

### visión general
- **Propósito**: devolver el meta‑modelo de la base de datos adaptado al usuario actual.
- **Campo GraphQL**: `Query.getModel` (tipo `String`). Devuelve un JSON en forma de cadena.
- **Resolver**: `graphql/resolvers/GetModelFetcher.java`.
- **Comportamiento**:
  - Usuarios con rol super reciben el modelo base completo con metadatos.
  - Usuarios no‑super reciben un modelo filtrado por privilegios y permisos de rol.

### dónde se registra el resolver
El resolver se vincula al campo `getModel` durante la construcción del esquema GraphQL.
```26:37:airflows-uno/com.niledb.platform/src/main/java/graphql/builders/SchemaModule.java
queryBuilder.field(newFieldDefinition()
  .name(GraphQLConstants.GET_MODEL_NAME)
  .description("It returns the system meta-data model.")
  .type(GraphQLString)
);
codeRegistryBuilder.dataFetcher(
  FieldCoordinates.coordinates(
    GraphQLConstants.QUERY_TYPE,
    GraphQLConstants.GET_MODEL_NAME
  ),
  new GetModelFetcher()
);
```

### ciclo de vida del esquema y activación del resolver
- El esquema se construye de forma perezosa la primera vez que se necesita y puede refrescarse posteriormente.
```79:87:airflows-uno/com.niledb.platform/src/main/java/graphql/GraphQLBuilder.java
public static synchronized GraphQL getGraphQL() {
  if (graphql == null) {
    refreshSchema();
  }
  return graphql;
}
```
- Cada petición HTTP GraphQL se maneja en `GraphQLHandler`, que crea el `ExecutionInput` con el contexto del usuario y delega en el esquema activo.
```66:76:airflows-uno/com.niledb.platform/src/main/java/graphql/GraphQLHandler.java
executor.executeBlocking(
    () -> GraphQLBuilder.getGraphQL().execute(executionInput),
    false
).onSuccess(result -> {
  JsonObject json = new JsonObject();
  ...
  json.put("data", result.getData());
  routingContext.response()...end(json.encode());
});
```
- Solo el campo `Query.getModel` pasa por `GetModelFetcher`. El resto de campos/operaciones usan otros resolvers.

### punto de entrada del resolver y extracción de contexto
```20:31:airflows-uno/com.niledb.platform/src/main/java/graphql/resolvers/GetModelFetcher.java
@Override
public String get(DataFetchingEnvironment environment) {
  AirflowsRequestContext afCtx = GraphQLUtils.getAirflowsRequestContext(environment);
  boolean rolsuper = afCtx.isSuper();
  boolean agreementAccepted = afCtx.isAgreementAccepted();
  String username = afCtx.getUsername();
  String email = afCtx.getEmail();
  ...
}
```
Datos clave del contexto:
- `isSuper`, `isAnonymous`, `agreementAccepted`, `username`, `email`.

### flujo detallado del resolver
1. **Conexión a BD**: abre conexión JDBC con `helpers/DatabaseHelper.getConnection()`.
2. **Resolución de valores por defecto y flags de rol**:
   - `defaultEntity`: desde `Models.Role.defaultEntity` → `Models.Entity` (por roles del usuario o rol `'anonymous'`).
   - `defaultExternalEntity`: análogo con `Models.ExternalEntity`.
   - `disableReferenceWidgetButtons`: desde `Models.Role.disableReferenceWidgetButtons` (por rol/roles vigentes).
3. **Rama por tipo de usuario**:
   - **Super**:
     - Carga todos los dashboards (`Models.Dashboard`).
     - Devuelve el modelo base completo (`ModelHelper.model`) enriquecido con metacampos (ver más abajo) y `dashboards`.
     ```117:129:airflows-uno/com.niledb.platform/src/main/java/graphql/resolvers/GetModelFetcher.java
     return ModelHelper.model
       .put("edition", ConfigHelper.get(ConfigHelper.EDITION, "Enterprise"))
       .put("super", rolsuper)
       .put("defaultEntity", defaultEntity)
       .put("defaultExternalEntity", defaultExternalEntity)
       .put("disableReferenceWidgetButtons", disableReferenceWidgetButtons)
       .put("agreementAccepted", agreementAccepted)
       .put("username", username)
       .put("email", email)
       .put("dashboards", dashboards)
       .encode();
     ```
   - **No‑super**:
     - Construye un resultado inicial copiando de `ModelHelper.model` solo `name`, `customTypes`, `enumTypes`, `documentation`, más `schemaNames` y `entities` vacíos.
     - **Privilegios de columna**: `information_schema.column_privileges` (roles del usuario o `'anonymous'`). Por cada fila añade `privilege_type` tanto a `entity.privileges` como a `attribute.privileges` y registra el `schema` en `result.schemaNames`.
     - **Privilegio DELETE** (nivel tabla): `information_schema.table_privileges` filtrado a `DELETE`, se añade a `entity.privileges`.
     - **Privilegio MENU**: `Models.EntityPermission` (unnest de `types`) vía roles, añade `"MENU"` a `entity.privileges`.
     - **Dashboards**: permitidos por `Models.DashboardPermission` vía roles (o `'anonymous'`).
     - **Entidades externas**: permitidas por `Models.ExternalEntityPermission` vía roles (o `'anonymous'`).
     - No hay lógica adicional para propagar flags: los atributos ya traen `isMultipleReference` desde el modelo base.
     - Devuelve el JSON con metacampos + `dashboards`, `externalEntities`, `username`, `email`.
     ```322:327:airflows-uno/com.niledb.platform/src/main/java/graphql/resolvers/GetModelFetcher.java
     return result
       .put("username", username)
       .put("email", email)
       .put("dashboards", dashboards)
       .put("externalEntities", externalEntities)
       .encode();
     ```

### el modelo base `ModelHelper.model`
- `ModelHelper.updateModel(Database)` serializa un `data.Database` (con Jackson) en un `JsonObject` de Vert.x.
```36:41:airflows-uno/com.niledb.platform/src/main/java/helpers/ModelHelper.java
public static void updateModel(Database database) {
  ModelHelper.database = database;
  model = new JsonObject(objectMapper.writeValueAsString(database));
}
```
- `Database` se construye en `helpers/DatabaseHelper.getDatabaseModel(...)` a partir de metadatos JDBC y algunas tablas de `Models.*`.
  - Campos superiores:
    - `name`: nombre de BD; `documentation`: "Model generated from Database '…'";
    - `schemaNames`: todos los esquemas salvo filtros (`pg_%`, `information_schema`, `cron`, `iam`, `custom`, `timescaledb` internos).
  - `entities["schema.tabla"]`:
    - `schema`, `name`, `type` (TABLE/FOREIGN TABLE/VIEW/MATERIALIZED VIEW), `documentation`.
    - `attributes`: nombre, tipo (mapeo incluyendo dominios→base), `array`, `required`, `defaultValue`, `length/precision/scale`, `encrypted` (desde `Models.EntityAttribute.isEncrypted`), `isMultipleReference`.
    - `keys`: claves primarias e índices (marca `textSearch` si el índice usa `to_tsvector`).
    - `references`: claves foráneas con enlace a la clave referenciada.
  - `customTypes` y `enumTypes`:
    - Tipos a partir de JDBC (`TYPE`), atributos con `getColumns`.
    - Enums creados bajo demanda; valores vía `SELECT enum_range(NULL::"schema"."type")`.

### cómo se construye el nodo `references` de cada entidad
- Se genera en el modelo base a partir de las claves foráneas de la BD, usando `DatabaseMetaData.getImportedKeys` para cada entidad.
- Para cada fila (parte de una FK, incluyendo compuestas):
  - Se agrupa por nombre de FK (`fk_name`) para construir una referencia por restricción.
  - Se establece `name` y `documentation` de la referencia.
  - Se resuelve la entidad referenciada y se enlaza la clave referenciada por nombre (`pk_name`).
  - Se añade el atributo local (columna FK) al listado ordenado de `attributes` de la referencia (el orden es importante en claves compuestas).
```770:807:airflows-uno/com.niledb.platform/src/main/java/helpers/DatabaseHelper.java
// Get references
for (String schemaName : schemaNames.keySet()) {
  rs = dbmd.getTables(catalogName, schemaName, null, new String[]{
    "TABLE",
    "FOREIGN TABLE",
    "VIEW",
    "MATERIALIZED VIEW"});

  while (rs.next()) {
    String tableName = (String) rs.getObject("table_name");

    Entity entity = entities.get(schemaName + "." + tableName);

    Map<String, EntityReference> references = entity.getReferences();
    Map<String, EntityAttribute> attributes = entity.getAttributes();

    ResultSet importedRs = dbmd.getImportedKeys(catalogName, schemaName, tableName);
    while (importedRs.next()) {
      String pkTableSchema = (String) importedRs.getObject("pktable_schem");
      String pkTableName = (String) importedRs.getObject("pktable_name");
      String fkColumnName = (String) importedRs.getObject("fkcolumn_name");
      String fkName = (String) importedRs.getObject("fk_name");
      String pkName = (String) importedRs.getObject("pk_name");

      EntityReference reference = references.get(fkName);
      if (reference == null) {
        reference = dataFactory.createEntityReference();
        reference.eSetContainer(entity);
        references.put(fkName, reference);
        reference.setName(fkName);
        reference.setDocumentation(
          "Model generated from Reference (foreign key) '" + fkName + "'");
        Entity referencedEntity = entities.get(
          pkTableSchema + "." + pkTableName);
        reference.setReferencedKey(referencedEntity.getKeys().get(pkName));
      }
      reference.getAttributes().add(attributes.get(fkColumnName));
    }
  }
  rs.close();
}
```
Además:
- Se hace un SELECT único a `Models.EntityReference` para precargar `isMultipleReference` por referencia.
- Durante la construcción de referencias:
  - Se marca `reference.isMultipleReference` según ese mapa precargado.
  - Se refleja el flag en cada atributo FK de la referencia con `attribute.isMultipleReference = true`.

En la respuesta `getModel`:
- **Super**: `entity.references` y `entity.attributes[*].isMultipleReference` llegan tal cual del modelo base.
- **No‑super**: al copiar cada entidad base, se vacían solo `attributes` y `privileges`, pero los atributos heredados ya incluyen `isMultipleReference` sin necesidad de propagación adicional.

### forma del resultado (JSON) y diferencias por rol
- **Tipo devuelto**: siempre `String` (se hace `.encode()` sobre el `JsonObject`).
- **Campos comunes**: `edition: string`, `super: boolean`, `defaultEntity: string|null`, `defaultExternalEntity: string|null`, `disableReferenceWidgetButtons: boolean`, `agreementAccepted: boolean`, `username: string`, `email: string`, `dashboards: Array<{ id:number, schema:string, name:string }>`.
- **Super**: incluye todo `ModelHelper.model` (con `schemaNames`, `entities`, `customTypes`, `enumTypes`, `documentation`).
- **No‑super**:
  - `name: string`, `schemaNames: { [schema:string]: string }` (solo esquemas presentes en privilegios),
  - `entities: { [schema.tabla]: Entity }` filtradas por privilegios:
    - `Entity.privileges: { [tipo:string]: string }` (agrega tipos de privilegio + `DELETE` + `MENU`).
    - `Entity.attributes[nombre].privileges: { [tipo:string]: string }` desde privilegios de columna.
  - `customTypes`, `enumTypes`, `documentation`.
  - `externalEntities: Array<{ id:number, schema:string, name:string }>`.

### mapeo de campos → origen de datos
- `edition`: `ConfigHelper.EDITION` (por defecto "Enterprise").
- `super`, `agreementAccepted`, `username`, `email`: `AirflowsRequestContext`.
- `defaultEntity`, `defaultExternalEntity`: `Models.Role` → (`defaultEntity`/`defaultExternalEntity`) → `Models.Entity` / `Models.ExternalEntity` (vía roles de usuario o `'anonymous'`).
- `disableReferenceWidgetButtons`: `Models.Role.disableReferenceWidgetButtons`.
- `privilegios de entidad/atributo`: `information_schema.column_privileges`.
- `DELETE` (tabla): `information_schema.table_privileges` (solo `'DELETE'`).
- `MENU`: `Models.EntityPermission` (unnest de `types`) vía roles.
- `dashboards`: `Models.Dashboard` (super) o `Models.DashboardPermission` (no‑super/anónimo) vía roles.
- `externalEntities`: `Models.ExternalEntityPermission` vía roles.
- Metadatos base de entidades/atributos/refs/keys: JDBC `DatabaseMetaData` + `Models.EntityAttribute.isEncrypted` + `enum_range(...)`.
 - `attribute.isMultipleReference`: `Models.EntityReference.isMultipleReference` precargado y reflejado en atributos durante `DatabaseHelper.getDatabaseModel(...)`.

### manejo de errores y cierre de recursos
- Ante excepción: se registra el error y se devuelve `""` (cadena vacía).
- La conexión JDBC se cierra en `finally`.

### notas de rendimiento y seguridad
- Usuarios no‑super implican varias consultas (privilegios de columna, privilegio DELETE, permiso MENU, dashboards, entidades externas) filtradas por roles.
- El modelo base (`ModelHelper.model`) se reutiliza; para no‑super se copia lo necesario, minimizando trabajo.
- El retorno como `String` evita construir un tipo GraphQL anidado, pero obliga al cliente a parsear JSON.

### archivos relevantes
- `airflows-uno/com.niledb.platform/src/main/java/graphql/resolvers/GetModelFetcher.java`
- `airflows-uno/com.niledb.platform/src/main/java/graphql/builders/SchemaModule.java`
- `airflows-uno/com.niledb.platform/src/main/java/graphql/GraphQLBuilder.java`
- `airflows-uno/com.niledb.platform/src/main/java/graphql/GraphQLHandler.java`
- `airflows-uno/com.niledb.platform/src/main/java/helpers/ModelHelper.java`
- `airflows-uno/com.niledb.platform/src/main/java/helpers/DatabaseHelper.java`


