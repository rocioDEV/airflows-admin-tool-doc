# Modo anónimo - Documentación técnica y funcional

## Índice
1. [Resumen ejecutivo](#resumen-ejecutivo)
2. [Casos de uso funcionales](#casos-de-uso-funcionales)
3. [Arquitectura técnica](#arquitectura-técnica)
4. [Implementación detallada](#implementación-detallada)
5. [Flujos de usuario](#flujos-de-usuario)
6. [Limitaciones y restricciones](#limitaciones-y-restricciones)
7. [Configuración y requisitos](#configuración-y-requisitos)
8. [Resolución de problemas](#resolución-de-problemas)
9. [Mejoras recientes](#mejoras-recientes)

## Resumen ejecutivo

El **modo anónimo** es una funcionalidad que permite a los usuarios acceder a las herramientas de administración de Airflows sin necesidad de autenticación mediante credenciales. Esta característica proporciona acceso de solo lectura o demostración a las funcionalidades del sistema, facilitando la evaluación y exploración sin comprometer la seguridad.

### Características principales:
- Acceso sin credenciales mediante botón "Acceder como invitado"
- Funcionalidad de solo lectura con permisos limitados
- Transición fluida entre modo anónimo y autenticado
- Soporte multiidioma (66+ idiomas)
- Gestión automática de sesiones y caché

## Casos de uso funcionales

### 1. Demostración del producto
- **Escenario**: Presentación del sistema a potenciales clientes
- **Beneficio**: Permite mostrar las capacidades sin requerir cuentas de prueba
- **Usuario objetivo**: Equipos de ventas, prospectos comerciales

### 2. Evaluación técnica
- **Escenario**: Evaluación de funcionalidades por parte de desarrolladores
- **Beneficio**: Acceso inmediato para revisar interfaces y flujos
- **Usuario objetivo**: Arquitectos de software, desarrolladores

### 3. Formación y capacitación
- **Escenario**: Entrenamientos internos o workshops
- **Beneficio**: Acceso rápido sin gestión de credenciales múltiples
- **Usuario objetivo**: Equipos de formación, nuevos empleados

### 4. Pruebas de concepto
- **Escenario**: Validación de workflows y procesos
- **Beneficio**: Exploración libre sin impacto en datos productivos
- **Usuario objetivo**: Analistas de negocio, consultores

## Arquitectura técnica

### Componentes principales

#### 1. Gestión de autenticación
**Archivo**: `src/contexts/useAuth.tsx`

```typescript
type AuthContextType = {
  accessToken: string | null
  anonymousLogin: boolean
  handleAnonymousLogin: () => void
  logout: () => void
}
```

**Funcionalidades**:
- Estado de sesión anónima persistente en localStorage
- Transición automática entre modos de autenticación
- Invalidación de caché al cambiar de modo
- Gestión de tokens y limpieza de estado

#### 2. Gestión de permisos
**Archivo**: `src/data/util/hasAnonymousPermissions.ts`

```typescript
export const hasAnonymousPermissions = (model: DBParsedModel | ErrorModel | undefined) => {
  return isParsedModel(model) && !isEmpty(model.entities)
}
```

**Lógica de activación**:
- Requiere modelo de base de datos válido
- Necesita al menos una entidad con datos
- Validación automática de disponibilidad

#### 3. Enrutamiento protegido
**Archivo**: `src/components/router/ProtectedRouter.tsx`

**Comportamiento**:
- Acepta tanto `accessToken` como `anonymousLogin`
- Redirección inteligente a página de login
- Preservación de URL de destino

#### 4. Gestión de consultas
**Archivo**: `src/components/bootstrap/all-models/useFetchModels.ts`

**Estrategia de consultas**:
- Claves de consulta separadas para modo anónimo
- `GET_MODEL_ANONYMOUS` vs `GET_MODEL`
- `GET_MODEL_DATA_ANONYMOUS` vs `GET_MODEL_DATA`
- Priorización de consultas autenticadas

### Flujo de datos

```mermaid
graph TD
    A[Usuario accede a ruta protegida] --> B{¿Autenticado?}
    B -->|No| C[Redirección a /login]
    C --> D[Página de login mostrada]
    D --> E{¿Clic en "Acceder como invitado"?}
    E -->|Sí| F[Verificar permisos anónimos]
    F --> G{¿Permisos disponibles?}
    G -->|Sí| H[Activar modo anónimo]
    G -->|No| I[Mostrar login normal]
    H --> J[Consultas con clave anónima]
    J --> K[Dashboard con funcionalidad limitada]
    E -->|No| L[Login con credenciales]
    L --> M[Token de acceso válido]
    M --> N[Consultas autenticadas]
    N --> O[Dashboard completo]
```

## Implementación detallada

### 1. Activación del modo anónimo

**Proceso**:
1. Usuario hace clic en "Acceder como invitado"
2. Sistema valida `hasAnonymousPermissions(model)`
3. Si es válido, ejecuta `handleAnonymousLogin()`
4. Se establece `anonymousLogin: true` en estado
5. Se guarda `'anonymous_login': 'true'` en localStorage
6. Se establece username como `'anonymous'`

**Código relevante**:
```typescript
const handleAnonymousLogin = useCallback(() => {
  setAnonymousLogin(true)
  setUsernamePersist('anonymous')
  afLocalStorage.setItem('anonymous_login', 'true')
}, [setUsernamePersist])
```

### 2. Gestión de consultas diferenciadas

**Lógica de selección**:
```typescript
const enabled = !!accessToken || anonymousLogin
const isAuthenticated = !!accessToken
const { data: model } = useQuery({
  queryKey: isAuthenticated ? GET_MODEL : GET_MODEL_ANONYMOUS,
  queryFn: () => getModel({ isMock: state.isMock }),
  enabled,
})
```

**Ventajas**:
- Cachés separados para cada modo
- Datos específicos según permisos
- Evita contaminación cruzada de datos

### 3. Transición entre modos

**Del modo anónimo al autenticado**:
1. Usuario hace clic en "Iniciar sesión" desde menú de usuario
2. Sistema ejecuta logout completo
3. Redirección a página de login
4. Usuario ingresa credenciales
5. Al establecer `accessToken`, se limpia automáticamente estado anónimo:

```typescript
if (anonymousLogin) {
  afLocalStorage.removeItem('anonymous_login')
  setAnonymousLogin(false)
  // Invalidar cachés anónimos
  queryClient.invalidateQueries({ queryKey: GET_MODEL_ANONYMOUS })
  queryClient.invalidateQueries({ queryKey: GET_MODEL_DATA_ANONYMOUS })
}
```

### 4. Interfaz de usuario adaptativa

**Menú de usuario**:
```typescript
{anonymousLogin ? <Login /> : <CloseOutlined />}
{t(anonymousLogin ? 'login' : 'closeSession')}
```

**Características**:
- Iconos adaptativos según modo
- Texto contextual
- Funcionalidad diferenciada

## Flujos de usuario

### Flujo 1: Acceso anónimo directo
1. Usuario visita URL protegida (ej: `/admin`)
2. Sistema detecta falta de autenticación
3. Redirección a `/login`
4. Usuario ve página de login con opción "Acceder como invitado"
5. Clic en botón de invitado
6. Acceso inmediato al dashboard con permisos limitados

### Flujo 2: Transición de anónimo a autenticado
1. Usuario en modo anónimo
2. Clic en botón "Iniciar sesión" en menú de usuario
3. Logout automático y redirección a login
4. Ingreso de credenciales válidas
5. Acceso completo con todos los permisos
6. Menú y funcionalidades completas disponibles

### Flujo 3: Persistencia de sesión
1. Usuario activa modo anónimo
2. Cierra navegador
3. Reabre aplicación
4. Sesión anónima se mantiene activa
5. Acceso directo al dashboard

## Limitaciones y restricciones

### 1. Funcionalidades restringidas
- **Actualización de tokens**: Deshabilitada completamente
- **Diálogos de acuerdo**: No se muestran para usuarios anónimos
- **Gestión de entidades**: Solo consultas `GET_MODEL_ENTITY_ID` requieren token
- **Funciones administrativas**: Acceso limitado según permisos del modelo

### 2. Restricciones de datos
- **Dependiente de entidades**: Solo funciona si existen entidades en el modelo
- **Permisos del modelo**: Limitado por la configuración del modelo de base de datos
- **Sin persistencia de cambios**: Modo efectivamente de solo lectura

### 3. Limitaciones técnicas
- **Headers de autorización**: No se envían en modo anónimo
- **Endpoints separados**: Usa endpoints específicos para usuarios anónimos
- **Caché independiente**: Cachés separados pueden causar inconsistencias temporales

## Configuración y requisitos

### Requisitos del sistema
1. **Modelo de base de datos válido**: Debe existir y ser parseable
2. **Entidades con datos**: Al menos una entidad debe contener información
3. **Configuración de backend**: Endpoints anónimos deben estar habilitados
4. **Permisos de modelo**: Configuración adecuada de permisos de lectura

### Variables de configuración
- **localStorage keys**: 
  - `anonymous_login`: Estado de sesión anónima
  - `username`: Se establece como 'anonymous'
- **Query keys**:
  - `GET_MODEL_ANONYMOUS`
  - `GET_MODEL_DATA_ANONYMOUS`

### Internacionalización
Soporte completo en 66+ idiomas:
- Español: "Acceder como invitado"
- Inglés: "Login as guest"
- Francés: "Connexion en tant qu'invité"
- Alemán: "Als Gast anmelden"
- Y muchos más...

## Resolución de problemas

### Problema 1: Botón de invitado no aparece
**Causa**: `hasAnonymousPermissions` retorna `false`
**Solución**: Verificar que el modelo tenga entidades con datos
**Diagnóstico**:
```typescript
// Verificar en consola del navegador
console.log(model?.entities) // Debe contener al menos una entidad
```

### Problema 2: Doble login requerido
**Causa**: Race condition entre estados de autenticación
**Solución**: Implementada limpieza automática de estado anónimo
**Verificación**: Comprobar que no existan tokens y flags anónimos simultáneamente

### Problema 3: Datos obsoletos después de cambio de modo
**Causa**: Cachés no invalidados correctamente
**Solución**: Invalidación automática implementada
**Verificación**: Comprobar que se ejecuten las invalidaciones de consultas

### Problema 4: Menú no aparece después de login
**Causa**: Estado de autenticación inconsistente
**Solución**: Priorización de consultas autenticadas
**Verificación**: Confirmar que `isAuthenticated` tenga prioridad sobre `anonymousLogin`

## Mejoras recientes

### Corrección de transición de modos (Enero 2025)
**Problema identificado**: 
- Race condition entre modo anónimo y autenticado
- Doble login requerido
- Menú no aparecía correctamente

**Soluciones implementadas**:
1. **Limpieza inmediata de estado**: Al establecer token de acceso, se limpia automáticamente el estado anónimo
2. **Invalidación de caché**: Se invalidan las consultas anónimas al cambiar a modo autenticado
3. **Priorización de consultas**: Las consultas autenticadas tienen prioridad sobre las anónimas
4. **Eliminación de auto-login**: Se removió el comportamiento de login automático anónimo

**Archivos modificados**:
- `src/contexts/useAuth.tsx`: Gestión mejorada de transiciones
- `src/components/bootstrap/all-models/useFetchModels.ts`: Priorización de consultas
- `src/components/router/ProtectedRouter.tsx`: Eliminación de flag automático
- `src/components/login/LoginContainer.tsx`: Simplificación de lógica de login

**Resultado**: Transición fluida y sin problemas entre modos de autenticación.

---

## Conclusiones

El modo anónimo proporciona una funcionalidad valiosa para demostraciones, evaluaciones y acceso de solo lectura al sistema Airflows. La implementación actual es robusta, con gestión adecuada de estados, cachés diferenciados y transiciones fluidas entre modos de autenticación.

Las mejoras recientes han resuelto los problemas de race conditions y estado inconsistente, proporcionando una experiencia de usuario óptima tanto para el acceso anónimo como para la transición a modo autenticado.

**Fecha de última actualización**: Enero 2025
**Versión del documento**: 1.0
**Autor**: Sistema de documentación automática
