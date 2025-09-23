# Documentación funcional

## Descripción general de la aplicación

**Agentes AI de Airflows** es una aplicación de IA conversacional que proporciona una interfaz de chat para interactuar con varios modelos y agentes de IA. La aplicación admite múltiples proveedores de IA (OpenAI, Google, Ollama) y permite a los usuarios tener conversaciones con agentes de IA que pueden ejecutar herramientas y funciones.

## Interfaz de inicio

### Pantalla de bienvenida
- **Punto de entrada:** Ruta `/assistant`
- **Estado inicial:** Interfaz de chat vacía con diálogo de bienvenida
- **Integración del usuario:** 
  - Aparece un diálogo de solicitud de nombre de usuario para nuevos usuarios
  - Nombre de usuario predeterminado: "Anonymous" si el usuario omite
  - Experiencia personalizada basada en el nombre ingresado

## Flujos de usuario y recorridos

### 1. Flujo de integración de nuevo usuario
```
1. Usuario visita /assistant
2. Aparece diálogo de bienvenida
3. Usuario ingresa nombre (opcional)
4. Se cierra el diálogo, se carga la interfaz de chat
5. Usuario puede comenzar conversación
```

### 2. Flujo de conversación de chat
```
1. Usuario selecciona Agente AI desde el menú desplegable
2. Usuario escribe mensaje en campo de entrada
3. Mensaje enviado a API del backend
4. IA procesa mensaje con herramientas/contexto
5. Respuesta transmitida de vuelta al usuario
6. Historial de mensajes guardado localmente
7. Usuario puede continuar conversación
```

### 3. Flujo de gestión de modelos
```
1. Usuario hace clic en "Pull Model" en configuración
2. Aparece diálogo de entrada de nombre de modelo
3. Usuario ingresa nombre del modelo (ej., "llama2")
4. Backend descarga modelo desde biblioteca de Ollama
5. Barra de progreso muestra estado de descarga
6. Modelo disponible para selección en menú de agentes
```

### 4. Flujo de gestión de historial de chat
```
1. Usuario hace clic en botón "New Chat"
2. Nueva sesión de chat creada con ID único
3. Chats anteriores listados en barra lateral
4. Usuario puede cambiar entre chats
5. Usuario puede eliminar chats individuales
6. Historial de chat persistido en IndexedDB
```

### 5. Flujo de gestión de configuración
```
1. Usuario hace clic en icono de configuración
2. Aparece menú desplegable de configuración
3. Usuario puede cambiar nombre de usuario
4. Usuario puede descargar nuevos modelos
5. Usuario puede alternar tema (claro/oscuro)
6. Configuración guardada en almacenamiento local
```

## Dominios y secciones

### 1. Autenticación y gestión de usuarios
**Propósito:** Manejar identidad y preferencias del usuario
**Componentes:**
- `UsernameForm` - Entrada y validación de nombre de usuario
- `EditUsernameForm` - Edición de nombre de usuario
- `UserSettings` - Gestión de configuración de usuario
- `useChatStore` - Gestión de estado del usuario

**Lógica de negocio:**
- Validación y persistencia de nombre de usuario
- Manejo de usuarios anónimos
- Almacenamiento de preferencias de usuario
- Gestión de sesiones

### 2. Interfaz de chat
**Propósito:** Interfaz principal de conversación
**Componentes:**
- `Chat` - Componente principal de chat
- `ChatList` - Visualización de mensajes
- `ChatMessage` - Renderizado de mensaje individual
- `ChatBottombar` - Área de entrada
- `ChatTopbar` - Selección de agente y controles

**Lógica de negocio:**
- Transmisión y visualización de mensajes
- Manejo de conversaciones en tiempo real
- Gestión de historial de mensajes
- Validación y procesamiento de entrada

### 3. Gestión de agentes AI
**Propósito:** Manejar selección y configuración de modelos de IA
**Componentes:**
- Selector de agente desplegable
- Interfaz de descarga de modelos
- Gestión de contexto de agente

**Lógica de negocio:**
- Recuperación de metadatos de agente desde GraphQL
- Selección de proveedor de modelo (OpenAI, Google, Ollama)
- Configuración de herramientas y contexto
- Descarga y gestión de modelos

### 4. Sistema de ejecución de herramientas
**Propósito:** Ejecutar herramientas y funciones de agentes AI
**Componentes:**
- Construcción dinámica de herramientas
- Manejo de parámetros de función
- API de ejecución de herramientas

**Lógica de negocio:**
- Generación de esquema de herramientas desde GraphQL
- Validación de parámetros usando Zod
- Ejecución de funciones vía solicitudes HTTP
- Manejo de errores y procesamiento de respuestas

### 5. Persistencia de datos
**Propósito:** Almacenar historial de chat y datos de usuario
**Componentes:**
- Almacenamiento IndexedDB
- Gestión de estado Zustand
- Almacenamiento local para preferencias

**Lógica de negocio:**
- Persistencia de sesiones de chat
- Almacenamiento de historial de mensajes
- Caché de preferencias de usuario
- Sincronización de datos

### 6. Manejo de medios
**Propósito:** Manejar adjuntos de imágenes y contenido visual
**Componentes:**
- `ImageEmbedder` - Interfaz de carga de imágenes
- `CodeDisplayBlock` - Resaltado de sintaxis de código
- `DynamicChart` - Renderizado de gráficos

**Lógica de negocio:**
- Procesamiento de imágenes Base64
- Adjunto de imágenes a mensajes
- Formateo de bloques de código
- Visualización de datos de gráficos

## Lógica de negocio

### 1. Procesamiento de agentes AI
```typescript
// Selección y configuración de agente
const agent = await fetchAgentFromGraphQL(selectedModel);
const tools = buildToolsFromAgent(agent.tools);
const model = createModelProvider(agent.provider, apiKey);
const context = buildSystemPrompt(agent.contexts);
```

### 2. Pipeline de procesamiento de mensajes
```typescript
// Flujo de mensajes
1. Validación de entrada del usuario
2. Formateo de mensaje con adjuntos
3. Solicitud API con configuración de agente
4. Ejecución de herramientas si es necesario
5. Transmisión de respuesta
6. Persistencia de mensaje
7. Actualización de UI
```

### 3. Sistema de ejecución de herramientas
```typescript
// Flujo de ejecución de herramientas
1. Generación de esquema de herramientas desde GraphQL
2. Validación de parámetros usando Zod
3. Solicitud HTTP al endpoint de función
4. Procesamiento de respuesta
5. Manejo de errores
6. Formateo de resultados
```

### 4. Gestión de estado
```typescript
// Persistencia de estado
- Sesiones de chat en IndexedDB
- Preferencias de usuario en localStorage
- Estado en tiempo real en Zustand
- Claves API y tokens en contexto
```

### 5. Manejo de errores
```typescript
// Escenarios de error
- Problemas de conectividad de red
- Fallos de validación de clave API
- Errores de carga de modelos
- Fallos de ejecución de herramientas
- Errores de procesamiento de mensajes
```

## Características principales

### 1. Soporte multi-proveedor
- **OpenAI:** Modelos GPT vía API de OpenAI
- **Google:** Modelos Gemini vía Google AI
- **Ollama:** Modelos locales vía Ollama
- **Personalizado:** Modelos compatibles con vLLM

### 2. Integración de herramientas
- Carga dinámica de herramientas desde GraphQL
- Validación y ejecución de parámetros
- Capacidades de llamada de funciones
- Manejo de errores y procesamiento de respuestas

### 3. Comunicación en tiempo real
- Transmisión de respuestas
- Actualizaciones de mensajes en vivo
- Indicadores de progreso
- Notificaciones de estado

### 4. Soporte de medios
- Adjuntos de imágenes
- Resaltado de sintaxis de código
- Visualizaciones de gráficos
- Manejo de carga de archivos

### 5. Persistencia
- Almacenamiento de historial de chat
- Preferencias de usuario
- Gestión de sesiones
- Capacidad offline

## Arquitectura técnica

### Stack frontend
- **Framework:** Next.js 14 con React 18
- **Gestión de estado:** Zustand con persistencia IndexedDB
- **Componentes UI:** Radix UI con Tailwind CSS
- **Formularios:** React Hook Form con validación Zod
- **Gráficos:** Recharts para visualización de datos

### Integración backend
- **Rutas API:** Rutas API de Next.js
- **GraphQL:** Metadatos y configuración de agentes
- **AI SDK:** SDK de IA de Vercel para integración de modelos
- **Transmisión:** Eventos enviados por servidor para respuestas en tiempo real

### Flujo de datos
```
Entrada de usuario → Validación frontend → Solicitud API → 
Consulta GraphQL → Ejecución de herramientas → Procesamiento AI → 
Transmisión de respuesta → Actualización frontend → Persistencia de estado
```

## Consideraciones de seguridad

### 1. Gestión de claves API
- Claves API almacenadas en contexto de aplicación
- Transmisión segura vía headers
- Sin exposición del lado cliente

### 2. Autenticación
- Autenticación basada en tokens
- Headers de autorización para solicitudes API
- Gestión de sesiones

### 3. Privacidad de datos
- Almacenamiento local para datos de usuario
- Sin datos sensibles en URLs
- Protocolos de comunicación seguros

## Optimizaciones de rendimiento

### 1. Estrategia de caché
- IndexedDB para historial de chat
- LocalStorage para preferencias
- Caché en memoria para datos frecuentes

### 2. Transmisión
- Transmisión de respuestas en tiempo real
- Carga progresiva
- División optimizada de bundles

### 3. Gestión de estado
- Actualizaciones eficientes de estado
- Re-renderizado selectivo
- Persistencia optimizada

## Mejoras futuras

### 1. Características planificadas
- Capacidades de entrada/salida de voz
- Tipos avanzados de gráficos
- Sesiones de chat colaborativas
- Sistema de plugins para herramientas

### 2. Mejoras de escalabilidad
- Conexiones WebSocket
- Colaboración en tiempo real
- Estrategias avanzadas de caché
- Monitoreo de rendimiento

Esta documentación funcional proporciona una visión completa de la aplicación Agentes AI de Airflows, cubriendo todos los aspectos principales desde la interfaz de usuario hasta la lógica de negocio y la arquitectura técnica. 