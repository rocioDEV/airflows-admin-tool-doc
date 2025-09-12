# crear un nuevo endpoint

Este documento explica cómo extender la API basada en Vert.x que vive en `com.niledb.platform` con un **nuevo endpoint HTTP** o con un **verticle de background**.

---

## repaso de la estructura del proyecto

```
com.niledb.platform/src/main/java
├── verticles/              # unidades en tiempo de ejecución desplegadas por Vert.x
├── handlers/               # una clase por handler REST/utilitario
├── helpers/                # configuración y utilidades compartidas
├── db/                     # acceso reactivo a BD y hub de notificaciones
├── graphql/                # builders de esquema y resolvers
└── workflows/, tools/, …   # módulos de dominio
```

Verticles clave (rutas relativas a `src/main/java`):

- `verticles/MainVerticle.java` – bootstrap; despliega el resto.
- `verticles/HttpVerticle.java` – crea el servidor HTTP y **todas las rutas**.
- `verticles/DbNotifyVerticle.java` – puente PostgreSQL LISTEN/NOTIFY.
- `verticles/PeriodicTasksVerticle.java` – jobs en background programados.
- `verticles/MqttVerticle.java` – soporte MQTT opcional.

---

## cómo se despliegan los verticles

El despliegue ocurre dentro de **`verticles/MainVerticle.java`**:

```java
vertx.deployVerticle(DbNotifyVerticle.class.getName());
vertx.deployVerticle(ExtractorEventsVerticle.class.getName());
vertx.deployVerticle(HttpVerticle::new);          // punto de entrada HTTP
vertx.deployVerticle(PeriodicTasksVerticle.class);
vertx.deployVerticle(MqttVerticle.class);         // si MQTT está habilitado
```

Añade tu propio verticle insertando otra llamada a `deployVerticle`.

---

## dónde se declaran las rutas http

Cada ruta REST vive en **`verticles/HttpVerticle.java`**. Pequeño extracto:

```java
Router router = Router.router(vertx);

router.get("/model-export")          .handler(ModelExportHandler::execute);
router.post("/model-import")         .handler(BodyHandler.create())
                                     .blockingHandler(ModelImportHandler::execute);
router.post("/graphql")              .handler(BodyHandler.create())
                                     .handler(GraphQLHandler::execute);
```

Los proxys para `/dashboards/**`, `/applications/**` y `/auth/**` se montan mediante métodos internos auxiliares (`setupDashboards`, `setupApplications`, `setupIAM`).

---

## añadir un verticle de background

1. Crea la clase, por ejemplo **`verticles/MyNewWorkerVerticle.java`**:

   ```java
   package verticles;

   import io.vertx.core.AbstractVerticle;

   public class MyNewWorkerVerticle extends AbstractVerticle {
       @Override
       public void start() {
           // lógica en background aquí
       }
   }
   ```

2. Regístralo en `MainVerticle.start()`:

   ```java
   vertx.deployVerticle(MyNewWorkerVerticle.class.getName())
        .onSuccess(id -> log.info("MyNewWorkerVerticle deployed: {}", id));
   ```

   Controla instancias/config con `new DeploymentOptions()` si lo necesitas.

---

## añadir un nuevo endpoint http

1. **Crea un handler** dentro del paquete `handlers`:

   **`handlers/ReportExportHandler.java`**

   ```java
   package handlers;

   import io.vertx.ext.web.RoutingContext;

   public class ReportExportHandler {
       public static void execute(RoutingContext ctx) {
           // leer parámetros / llamar servicios / escribir respuesta
           ctx.response().end("ok");
       }
   }
   ```

2. **Registra la ruta** en `HttpVerticle.start()` (cerca de las similares):

   ```java
   router.get("/report-export")
         .handler(ReportExportHandler::execute);
   // si necesitas body:
   router.post("/report-export")
         .handler(BodyHandler.create())
         .blockingHandler(ReportExportHandler::execute);
   ```

   Usa `blockingHandler` cuando las operaciones sean intensivas en CPU o I/O bloqueante.

3. Recompila/ejecuta. No hace falta más cableado porque `MainVerticle` ya despliega `HttpVerticle`.

---

## tips y buenas prácticas

- Configuración global de CORS y BasicAuth opcional se aplican automáticamente en `HttpVerticle`.
- Las claves de configuración viven en **`helpers/ConfigHelper.java`** y en el `config.json` raíz.
- Usa **`db/core/ReactiveDbClient`** para acceso a base de datos no bloqueante.
- Mantén los handlers livianos; delega trabajo pesado a verticles de worker o servicios.
- Para operaciones de larga duración, prefiere `router.post(...).blockingHandler(...)`.
- Recuerda añadir tests o verificaciones manuales con curl para el nuevo endpoint.


