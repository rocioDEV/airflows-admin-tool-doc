### Sistema de flags de características (/airflows-features)

Esta página describe el sistema de feature flags de Airflows y el endpoint `/airflows-features` que expone el backend para que el frontend active o desactive funcionalidades de forma dinámica.

### Qué es

- **Objetivo**: permitir habilitar/deshabilitar áreas del producto sin desplegar código nuevo.
- **Endpoint**: `GET /airflows-features` devuelve un JSON con flags booleanos.
- **Flags actuales**: `WORKFLOWSV2`, `AIV2`.

### Backend

- **Ruta**: implementada en `com.niledb.platform/src/main/java/verticles/HttpVerticle.java`.
- **Respuesta**: JSON con claves de flags y valores booleanos.

```json
{
  "WORKFLOWSV2": true,
  "AIV2": true
}
```

- **Implementación** (extracto):

```java
router.get("/airflows-features").handler(ctx -> ctx.response()
  .putHeader("Content-Type", "application/json")
  .end(
    new JsonObject()
      .put("WORKFLOWSV2",  ConfigHelper.get(ConfigHelper.WORKFLOWSV2_ENABLED, false))
      .put("AIV2", ConfigHelper.get(ConfigHelper.AIV2_ENABLED, false))
      .encode()
  )
);
```

- **Claves de configuración**:
  - `WORKFLOWSV2_ENABLED`
  - `AIV2_ENABLED`

- **Valores por defecto en el repositorio**:
  - `com.niledb.platform/config.json`: ambos en `true`.
  - `com.niledb.platform/misc/docker/config.json`: ambos en `true`.

- **Seguridad y proxy**:
  - La ruta está en `excludedExactPaths`, por lo que siempre se sirve localmente (no pasa por proxy interno del servidor).
  - Si `SERVICE_AUTHENTICATE` está activo, se aplicará autenticación básica global; si no, el endpoint es legible sin credenciales.

- **cURL de ejemplo**:

```bash
curl -s https://<host>/airflows-features
```

### Frontend

- **Tipo y consulta**: `src/data/queries/getAirflowsFeatures.ts`

```ts
export type AirflowsFeatures = {
  WORKFLOWSV2?: boolean
  AIV2?: boolean
}

export const getAirflowsFeatures = async (): Promise<AirflowsFeatures> => {
  const { baseUrl } = getAppConfig()
  try {
    const response = await axios<AirflowsFeatures>(baseUrl + `/airflows-features`, { method: 'GET' })
    return response.data
  } catch (e) {
    return {}
  }
}
```

- **Consumo y efecto**: `src/components/bootstrap/all-models/useFetchModels.ts`
  - Se consulta siempre mediante React Query (`GET_AIRFLOWS_FEATURES`).
  - Los flags influyen en el bootstrap de datos iniciales, fusionando datasets locales:
    - Si `AIV2` → se fusiona `aiaAllData.json`.
    - Si `WORKFLOWSV2` → se fusiona `workflowsAllData.json`.
  - Los flags se guardan en el estado global (`useAppStateContext`) como `state.airflowsFeatures` y se propagan al resto de la aplicación.

```ts
let allModelDataMerge = allModelData
if (airflowsFeatures?.AIV2) {
  allModelDataMerge = mergeAllData(allModelDataMerge, aiaAllData)
}
if (airflowsFeatures?.WORKFLOWSV2) {
  allModelDataMerge = mergeAllData(allModelDataMerge, workflowsAllData)
}
```

### cómo añadir un nuevo flag

1) **Backend**
- Añadir la constante del nuevo flag en `ConfigHelper.java` (por ejemplo, `NEWFLAG_ENABLED`).
- Definir su valor por defecto en `com.niledb.platform/config.json` (y en la variante Docker si aplica).
- Extender el JSON del handler de `/airflows-features` en `HttpVerticle` con `.put("NEWFLAG", ConfigHelper.get(ConfigHelper.NEWFLAG_ENABLED, false))`.

2) **Frontend**
- Extender el tipo `AirflowsFeatures` con `NEWFLAG?: boolean`.
- Consumir el flag donde corresponda (UI, carga de datos, etc.).

3) **Documentación**
- Actualizar esta página con la descripción del nuevo flag y su efecto.

### solución de problemas

- **404 en `/airflows-features`**: verificar que el servicio está levantado y que no hay reglas de proxy externas bloqueando la ruta.
- **401/403**: si `SERVICE_AUTHENTICATE` está activo, incluir credenciales válidas en la petición.
- **El cambio de un flag no se refleja en el UI**: el frontend consulta en el arranque; fuerza recarga de la página y/o invalida la caché correspondiente si se está aplicando caché de red inversa.

### notas

- Actualmente los flags se usan para enriquecer datos de inicio (AI v2 y Workflows v2). No hay otras referencias directas adicionales en el admin tool.


