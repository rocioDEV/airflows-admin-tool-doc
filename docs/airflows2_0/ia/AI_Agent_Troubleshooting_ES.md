# Resolución de problemas del AI Agent con Docker

## Resumen del problema

Se encontraron múltiples problemas al configurar el AI Agent en contenedores Docker, desde problemas de conectividad de red hasta configuraciones SSL y endpoints faltantes.

## Problemas identificados y soluciones

### 1. Variable de entorno no visible en PL/Python

**Problema:**
```
AI Agent Task 'New Node' could not be completed due to 'HTTPSConnectionPool(host='localhost', port=8443): Max retries exceeded with url: /graphql (Caused by NewConnectionError('<urllib3.connection.HTTPSConnection object at 0x7f8f04d28160>: Failed to establish a new connection: [Errno 111] Connection refused'))'
```

**Causa:**
- La variable `AIRFLOWS_INSTANCE_URL` estaba configurada en el contenedor Docker
- Pero `get_env_variable()` en PL/Python lee desde la tabla `Models.Variable`, no desde las variables de entorno del contenedor
- La función fallaba al intentar conectarse a `localhost:8443` (valor por defecto)

**Solución:**
```sql
-- Insertar la variable en la tabla de la base de datos
INSERT INTO "Models"."Variable" ("name","value")
VALUES ('AIRFLOWS_INSTANCE_URL','https://host.docker.internal:8443')
ON CONFLICT ("name") DO UPDATE
SET "value" = EXCLUDED."value";
```

### 2. Resolución DNS fallida en contenedores Linux

**Problema:**
```
NameResolutionError ... Failed to resolve 'host.docker.internal'
```

**Causa:**
- `host.docker.internal` no existe por defecto en Docker Linux
- El contenedor PostgreSQL no podía resolver el nombre del host

**Solución:**
```yaml
# En docker-compose.yml
services:
  db:
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

O usar la IP del gateway Docker:
```bash
echo "172.17.0.1 host.docker.internal" | docker exec -i db2 tee -a /etc/hosts
```

### 3. Servicio backend no accesible desde contenedores

**Problema:**
```
Failed to connect to host.docker.internal port 8443 after 0 ms: Could not connect to server
```

**Causa:**
- El servicio Java estaba escuchando solo en `127.0.0.1:8443` (loopback)
- Los contenedores no pueden acceder a la interfaz loopback del host

**Solución:**
Configurar el servicio para escuchar en todas las interfaces:
```properties
# En application.properties o como variable de entorno
server.address=0.0.0.0
```

### 4. Problemas de construcción de Docker

**Problema:**
```
failed to calculate checksum ... "/misc/geoip/GeoLite2-City.mmdb": not found
```

**Causa:**
- El Dockerfile asume que se ejecuta desde `com.niledb.platform/`
- Pero `docker compose` se ejecuta desde la raíz del proyecto
- Los paths `COPY misc/...` no existen en el contexto de construcción

**Solución:**
```yaml
# En compose.yaml
services:
  backend:
    build:
      context: ./com.niledb.platform        # Contexto correcto
      dockerfile: misc/docker/Dockerfile
```

### 5. Extensiones PostgreSQL faltantes

**Problema:**
```
extension "pg_cron" is not available
```

**Causa:**
- La imagen `postgres:15` no incluye `pg_cron`
- El script de inicialización falla al intentar crear la extensión

**Solución:**
```yaml
# Usar imagen con pg_cron incluido
services:
  db:
    image: ghcr.io/citusdata/pg_cron:15
```

O crear imagen personalizada:
```dockerfile
FROM postgres:15
RUN echo "deb https://apt.postgresql.org/pub/repos/apt $(grep VERSION_CODENAME /etc/os-release | cut -d= -f2)-pgdg main" > /etc/apt/sources.list.d/pgdg.list \
 && apt-get update \
 && apt-get install -y postgresql-15-cron
```

### 6. Migraciones de base de datos no aplicadas

**Problema:**
```
ERROR: schema "Models" does not exist
```

**Causa:**
- El esquema base no se creó antes de intentar aplicar las migraciones delta
- El backend intenta leer `Models.Version` pero la tabla no existe

**Solución:**
```yaml
# Montar script de inicialización
services:
  db:
    volumes:
      - ./com.niledb.platform/misc/model/create.sql:/docker-entrypoint-initdb.d/00_create.sql
```

### 7. Problemas de autenticación de base de datos

**Problema:**
```
password authentication failed for user "postgres"
```

**Causa:**
- Las credenciales en `compose.yaml` no coinciden con las almacenadas en el cluster PostgreSQL
- Cambio de contraseña sin recrear el volumen

**Solución:**
```bash
# Opción 1: Recrear desde cero
docker compose down -v
docker compose up -d

# Opción 2: Cambiar contraseña en PostgreSQL
docker compose exec db psql -U postgres -d postgres
ALTER ROLE postgres WITH PASSWORD 'nuevaContraseña';
```

### 8. Problemas SSL/HTTPS

**Problema:**
```
AcmeServerException: Invalid identifiers ... Cannot issue for "0.0.0.0"
```

**Causa:**
- Let's Encrypt no puede emitir certificados para direcciones IP
- El backend intenta generar certificados automáticamente

**Solución:**
```yaml
# Deshabilitar SSL para desarrollo
services:
  backend:
    environment:
      SERVICE_SSL: "false"
      SERVICE_PORT: "80"
      SERVICE_SSL_AUTO_GENERATE_CERT: "false"
      SERVICE_FORCE_SSL: "false"
    ports:
      - "8443:80"    # Mapear puerto host 8443 → contenedor 80
```

### 9. Endpoint de API faltante

**Problema:**
```
Failed to connect to AI Agent API -> status code: 404
```

**Causa:**
- La función `aia.invokeAIAgent` intenta llamar a `/api/agent`
- Este endpoint no existe en el backend

**Solución:**
- Crear el endpoint `/api/agent` en el backend, o
- Modificar la función para usar el endpoint GraphQL existente: `/graphql`

## Configuración final recomendada

### docker-compose.yml
```yaml
version: "3.9"

services:
  db:
    image: ghcr.io/citusdata/pg_cron:15
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: Postgres1234!
      POSTGRES_DB: niledb
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d niledb"]
      interval: 5s
      retries: 10
      start_period: 5s
    volumes:
      - ./com.niledb.platform/misc/model/create.sql:/docker-entrypoint-initdb.d/00_create.sql
    extra_hosts:
      - "host.docker.internal:host-gateway"

  backend:
    build:
      context: ./com.niledb.platform
      dockerfile: misc/docker/Dockerfile
    depends_on:
      db:
        condition: service_healthy
    environment:
      DB_HOST: db
      DB_PORT: 5432
      DB_NAME: niledb
      DB_USERNAME: postgres
      DB_PASSWORD: Postgres1234!
      SERVICE_SSL: "false"
      SERVICE_PORT: "80"
      SERVICE_SSL_AUTO_GENERATE_CERT: "false"
      SERVICE_FORCE_SSL: "false"
    ports:
      - "8443:80"
```

### Variables de entorno necesarias
```bash
# En el contenedor PostgreSQL (si se necesita acceso externo)
AIRFLOWS_INSTANCE_URL=http://backend:80

# En la tabla Models.Variable
INSERT INTO "Models"."Variable" ("name","value")
VALUES ('AIRFLOWS_INSTANCE_URL','http://backend:80')
ON CONFLICT ("name") DO UPDATE
SET "value" = EXCLUDED."value";
```

## Comandos útiles para debugging

### Verificar estado de contenedores
```bash
docker compose ps
docker compose logs -f [service_name]
```

### Probar conectividad desde contenedor
```bash
# Desde contenedor db
docker compose exec db curl -v http://backend:80/graphql

# Verificar resolución DNS
docker compose exec db getent hosts backend
```

### Recrear servicios
```bash
# Recrear todo
docker compose down -v
docker compose up -d --build

# Recrear solo un servicio
docker compose up -d --build [service_name]
```

## Notas importantes

1. **Contexto de construcción**: El Dockerfile debe construirse desde `./com.niledb.platform/`
2. **Puertos**: El backend escucha en puerto 80 (HTTP) dentro del contenedor
3. **Red**: Los servicios se comunican usando nombres de servicio (`backend`, `db`)
4. **Migraciones**: El esquema base debe crearse antes de aplicar las migraciones delta
5. **SSL**: Para desarrollo, es más simple usar HTTP entre contenedores

## Estado final esperado

- ✅ Contenedor PostgreSQL ejecutándose con extensión `pg_cron`
- ✅ Backend ejecutándose en HTTP puerto 80
- ✅ Esquema `Models` creado con todas las tablas
- ✅ Migraciones delta 1-141 aplicadas exitosamente
- ✅ Conectividad entre contenedores funcionando
- ✅ Endpoint GraphQL accesible en `http://localhost:8443/graphql`
- ✅ Función `aia.invokeAIAgent` ejecutándose correctamente




--------------------


# troubleshooting ai workflow connectivity

## what went wrong

* The AI activity tried to POST to `/graphql` on **localhost** from inside the PostgreSQL container.
* No GraphQL server was listening in that container, so the connection was refused.  
  * First it failed on `http://localhost:5173/graphql` (default value).  
  * After changing the default it failed on `https://localhost:8443/graphql`.

## why it happened

1. `AiExecutorService` invokes a PostgreSQL function:

   ```
   select "aia"."invokeAIAgent"( … )
   ```

2. The PL/Python body of `aia.invokeAIAgent` does:

   ```python
   import os, requests
   url_base = os.getenv('AIRFLOWS_INSTANCE_URL', 'http://localhost:5173')
   requests.post(f'{url_base}/graphql', …)
   ```

3. If `AIRFLOWS_INSTANCE_URL` is **not** in the DB container’s environment,
   the code falls back to the hard-wired localhost URL, which is unreachable
   from inside the container.

## effective fix

* Supply a reachable URL to the PostgreSQL process through the
  `AIRFLOWS_INSTANCE_URL` environment variable.
* Because a running program cannot change its own environment, the database
  container must be **recreated or restarted** with the new variable.

Example:

```bash
docker stop db
docker rm db
docker create \
  --name db \
  -e POSTGRES_PASSWORD=Postgres1234! \
  -e AIRFLOWS_INSTANCE_URL=https://airflows-backend:8443 \
  -p 5432:5432 \
  hub.airflows.com:5443/airflows/postgres:15
docker start db
```

(or edit your `docker-compose.yml` / Kubernetes manifest accordingly).

## key takeaways

* The Java code does not connect to GraphQL directly; the call happens in a
  PL/Python function living in the database.
* Always set `AIRFLOWS_INSTANCE_URL` (or any similar variable) in the service
  that actually performs the HTTP request—in this case the PostgreSQL
  container.
* Changing environment variables for PostgreSQL requires a service restart.


---

# troubleshooting airflow-uno connection and migrations

This document summarizes the main findings and fixes discovered while debugging Airflows-uno on your development machine.

## 1. sql migration errors

1. `syntax error at or near "Completed"`  
   * Cause: A stray log line at the top of `misc/model/sql/deltas/139.sql`.  
   * Fix: Remove the line or comment it (`-- …`) so the file starts with valid SQL.

2. `schema "Workflows" does not exist`  
   * Cause: Migration 139 created enum types in schema `Workflows` before the schema existed.  
   * Fix: Prepend  
     ```sql
     CREATE SCHEMA IF NOT EXISTS "Workflows" AUTHORIZATION postgres;
     ```  
     to `139.sql`.

## 2. pl/python agent calling localhost:8443

* GraphQL endpoint is built from  

  ```python
  os.getenv("AIRFLOWS_INSTANCE_URL", "https://localhost:8443") + "/graphql"
  ```

* If `AIRFLOWS_INSTANCE_URL` is unset the agent falls back to `https://localhost:8443`.
* After setting the variable you must **restart the Postgres container** (or the whole stack) because pg-cron / background workers cache the environment.

## 3. exposing airflows on the public address

1. **Port-forwarding**: In the home router forward external `8443` (or `443`) → internal `192.168.1.100:8443`.
2. **Linux firewall**:  
   ```bash
   sudo ufw allow 8443/tcp        # or 443 if remapped
   ```
3. **Service configuration**  
   * In `config.json` / `misc/docker/config.json` adjust:
     ```json
     "SERVICE_HOST":   "92.190.101.70",
     "SERVICE_FQDN":   "92.190.101.70",
     "SERVICE_PORT":   8443,
     "SERVICE_SSL":    true
     ```
   * Provide a certificate that matches the public IP / DNS, or use `-k` with curl for testing.

4. **Environment variable for agents**  
   ```bash
   AIRFLOWS_INSTANCE_URL=https://92.190.101.70:8443
   ```

## 4. curl tests

* Port 8443 speaks HTTPS.  
  Use `curl -k https://192.168.1.100:8443` (the `-k` flag skips certificate
  validation).
* Plain `curl 192.168.1.100:8443` fails with  
  `curl: (52) Empty reply from server` because no TLS handshake occurs.

## 5. container → host communication

* Preferred hostname inside any Docker container:

  ```
  https://host.docker.internal:8443
  ```

  (works on Docker 20.10+ on Linux and all Docker Desktop installations).

* Fallback on older Linux daemons: use the bridge gateway (typically `172.17.0.1`).

## 6. json parsing exception in FunctionsApi

* Exception: `com.fasterxml.jackson.databind.exc.MismatchedInputException` \
  caused by empty `functionVariables`.
* Fix: guard the parse with

  ```java
  if (rawJson != null && !rawJson.isBlank()) {
      mapper.readValue(…);
  }
  ```

---

With these changes applied:

* Database migrations up to 141 execute cleanly.  
* The agent calls the public GraphQL endpoint instead of `localhost`.  
* External devices reach Airflows at `https://92.190.101.70:8443/`.