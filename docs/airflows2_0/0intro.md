---
sidebar_position: 0
---

# Introducción general de la arquitectura del front-end

Este documento sirve como punto de partida para el resto de la documentación técnica del proyecto. Resume dónde obtiene la aplicación sus datos, cómo se gestiona la autenticación y qué módulos de React constituyen los pilares de la interfaz.

## Fuentes de datos principales

### 1. `localModel` (modelo local)
* **Ubicación**: `src/data/mocks/localModel.ts` y `src/data/mocks/localModelJson.json`.
* **Propósito**: complementar y especializar el modelo que llega del servidor. Aquí se declaran:
  * propiedades de presentación (orden, visibilidad, iconos, estilo de entrada);
  * valores por defecto por atributo;
  * filtros adicionales, validaciones y reglas de negocio de la UI;
  * metadatos para _labels_ y columnas auxiliares.
* **Uso típico**:  
  El hook `useLoadEntityState` fusiona este modelo con el modelo de base de datos, de modo que la pantalla conozca tanto los requisitos del back-end como las preferencias de UX.

### 2. api GraphQL
* **Cliente**: `graphql-request` envuelto por `graphqlQueryRaw` (`src/data/queries/graphqlQueryRaw.ts`).
* **Consultas/mutaciones generadas**:  
  * estáticas en `src/data/queries/*` (por ejemplo, `getEntity.ts`, `backupExport.ts`), y  
  * automáticas en `src/generated/graphql.ts` mediante codegen (`codegen.yml`).
* **Capa de caché**: React-Query almacena los resultados para evitar redescargas y ofrece _invalidations_ simples.
* **Patrón de acceso**: las funciones de `src/data/queries` construyen _queries_ dinámicas, mientras que los componentes las consumen a través de hooks propios o de React-Query.

## Autenticación y ciclo de sesión
* **Tipo**: portador JWT en cabecera `Authorization`.
* **Gestión**:
  * `useAuth` (`src/contexts/useAuth.tsx`) centraliza el token y el nombre de usuario.
  * El token se persiste en `localStorage` bajo la clave `access_token`.
  * `AuthProvider` expone métodos para **login**, **logout** y **anonymousLogin**.
  * `authorizationHeaderHelper` inserta el token en cada petición GraphQL.
  * `useRefreshToken` lanza una mutación `refreshToken` cada minuto (o al volver a foco) para prolongar la sesión.
* **Flujo resumido**:
  1. El usuario se autentica en `/login` y recibe un JWT.
  2. El JWT se guarda en `localStorage` y se propaga por contexto.
  3. Cada consulta GraphQL incorpora la cabecera `Authorization: Bearer <JWT>`.
  4. Si el JWT va a caducar, `useRefreshToken` solicita uno nuevo al back-end.
  5. `logout` limpia el almacenamiento y fuerza redirección a `/login`.

## Componentes y dominios principales
La aplicación está organizada por dominios funcionales; cada dominio cuenta con componentes, hooks y tests propios. A continuación se muestran los más relevantes:

| Carpeta                                     | Rol principal | Piezas destacadas |
|---------------------------------------------|---------------|-------------------|
| `src/components/bootstrap`                  | Arranque global, carga de configuración, tema y refresco de token | `Bootstrap.tsx`, `useRefreshToken` |
| `src/components/dashboard`                  | Navegación y barra superior; selector de menú | `Dashboard.tsx`, `AirflowsAppBar`, `AirflowsMenuDrawer` |
| `src/components/entity-view`                | Formulario completo de visualización/edición de una entidad | `EntityViewContainer`, hooks `useEntityViewController`, `useLoadEntityState` |
| `src/components/entity-list`                | Listado, filtros y tabla de entidades | `EntityListContainer`, su árbol de filtros y toolbar |
| `src/components/entity-list-viewers`        | Visores alternativos (calendario, diagrama, galería, mapa) | `CalendarViewer`, `DiagramViewer`, `GalleryViewer`, `MapViewer` |
| `src/components/settings`                   | Pantallas de administración (dominios, BBDD, IAM, etc.) | Varios sub-módulos en `components/` |
| `src/components/login`                      | Flujos de inicio de sesión y cambio de contraseña | `LoginContainer`, `useLogin`, `useChangePassword` |
| `src/components/common` / `src/components/ui` | Controles genéricos reutilizables | `Questions`, `AutoComplete`, `AirflowsTabs`, `Barcode`, etc. |

La comunicación entre estos dominios se realiza mediante:
* **hooks contextuales** (`useAppStateContext`, `useAuth`, `useGlobalMessage`),
* **biblioteca de diseño** basada en _Material-UI_ y `emotion`,
* **React-Router** para enrutado protegido (`ProtectedRouter`).


## Gestión global de estado y mensajería
La comunicación entre partes de la aplicación se basa en **contextos de React**:

| Contexto | Ubicación | Responsabilidad |
|----------|-----------|-----------------|
| `AppStateContext` | `src/contexts/useAppStateContext.tsx` | Contiene el modelo completo (`state.model`), preferencias del usuario y timestamps de procesos. Ofrece selectores memorizados para acceder a metadatos de entidades. |
| `AuthContext` | `src/contexts/useAuth.tsx` | Mantiene el token JWT, nombre de usuario y funciones de login/logout/anonymousLogin; sincroniza el token con `localStorage`. |
| `GlobalMessageContext` | `src/contexts/useGlobalMessage` | Abstrae las notificaciones tipo _snackbar_ o _toast_; los componentes despachan mensajes globales sin acoplarse a la implementación visual. |

Estos contextos permiten un estado global ligero sin necesidad de Redux; se combinan con caches de **React Query** para los datos remotos.

## Sistema de diseño y temas personalizables
* **Material-UI** es la base visual; los estilos se generan con `@mui/material` y `@emotion/react`.
* `src/components/bootstrap/personalization/buildTheme.ts` construye un tema a partir de las preferencias almacenadas en el back-end (colores, tipografía, densidad).  
  El resultado se pasa al `ThemeProvider`, por lo que todo el árbol de componentes respeta esa configuración.
* Las variaciones de marca se almacenan en `AppStateContext.state.themeCustomization` y se pueden cambiar en caliente sin recargar la página.

## Internacionalización
* **Biblioteca**: `react-i18next` + `i18next-browser-languagedetector`.
* **Inicialización**: `src/components/i18n/i18n.ts` detecta idioma del navegador y carga recursos.
* **Recursos**: ficheros incrustados en `resources` (e.g. `en_US`, `es_ES`). Cada dominio puede añadir sus propias claves bajo `translation`.
* **Uso en componentes**: hook `useTranslation()` expone la función `t()`; los componentes pasan las claves definidas en los recursos.  
  Ejemplo:
  ```tsx
  const { t } = useTranslation();
  <Button>{t('save')}</Button>
  ```

## Testing
* **unitarios**: Vitest/Jest con `@testing-library/react` (ver carpetas `__tests__` y ficheros `*.spec.ts`).
* **end-to-end**: Playwright con escenarios en `e2e-tests/`; los resultados se vuelcan en Markdown dentro de `e2e-test-results/` con capturas de pantalla automáticas.
* **objetivo**: cubrir lógica de negocio, render de componentes críticos y flujos de usuario básicos (login, creación/edición/búsqueda de entidades).
