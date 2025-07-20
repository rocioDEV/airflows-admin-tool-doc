# Agentes AI de Airflows - Análisis de entradas de la aplicación

Este documento proporciona un análisis completo de todas las entradas a la aplicación Agentes AI de Airflows, categorizadas por tipo, fuente y propósito.

## 1. Entradas de interfaz de usuario

### 1.1 Entradas de texto

#### **Entrada de mensaje de chat**
- **Ubicación:** `src/components/ui/chat/chat-input.tsx`
- **Tipo:** Área de texto con auto-redimensionamiento
- **Propósito:** Entrada principal de conversación
- **Características:**
  - Área de texto auto-redimensionable
  - Texto de marcador: "Enter your prompt here"
  - Atajos de teclado (Enter para enviar, Shift+Enter para nueva línea)
  - Contador de caracteres en tiempo real
  - Gestión de foco

#### **Entrada de nombre de usuario**
- **Ubicación:** `src/components/username-form.tsx`
- **Tipo:** Entrada de texto con validación
- **Propósito:** Identificación del usuario
- **Validación:**
  - Mínimo 2 caracteres
  - Campo requerido
  - Validación de esquema Zod
- **Características:**
  - Marcador: "Enter your name"
  - Validación de formulario con mensajes de error
  - Persistencia en almacenamiento local

#### **Entrada de nombre de modelo**
- **Ubicación:** `src/components/pull-model-form.tsx`
- **Tipo:** Entrada de texto para selección de modelo
- **Propósito:** Descargar modelos de IA desde Ollama
- **Validación:**
  - Mínimo 1 carácter
  - Campo requerido
- **Características:**
  - Marcador: "llama2"
  - Seguimiento de progreso durante la descarga
  - Manejo de errores para descargas fallidas

### 1.2 Entradas de archivos

#### **Carga de imágenes**
- **Ubicación:** `src/components/image-embedder.tsx`
- **Tipo:** Carga de archivos con arrastrar y soltar
- **Propósito:** Adjuntar imágenes a mensajes de chat
- **Formatos Soportados:**
  - JPEG (.jpeg, .jpg)
  - PNG (.png)
  - GIF (.gif)
  - WebP (.webp)
  - PDF (.pdf)
- **Características:**
  - Selección múltiple de archivos
  - Interfaz de arrastrar y soltar
  - Límite de tamaño: 10MB por archivo
  - Conversión Base64 para transmisión API
  - Retroalimentación visual durante la carga

### 1.3 Entradas de voz

#### **Reconocimiento de voz**
- **Ubicación:** `src/app/hooks/useSpeechRecognition.ts`
- **Tipo:** API Web Speech del navegador
- **Propósito:** Conversión de voz a texto
- **Características:**
  - Transcripción en tiempo real
  - Soporte de idiomas (predeterminado: Español)
  - Modo de escucha continua
  - Reconocimiento de gramática
  - Manejo de errores para navegadores no soportados
  - Capitalización automática

## 2. Entradas de API

### 2.1 API de chat (`/api/chat`)

#### **Cuerpo de la Solicitud:**
```typescript
{
  messages: Message[],           // Historial de conversación
  selectedModel: string,         // Nombre del agente IA
  data?: {                      // Datos de imagen opcionales
    images: string[]            // Imágenes codificadas en Base64
  }
}
```

#### **Encabezados:**
```typescript
{
  "Authorization": "Bearer ${token}",
  "X-Openai-API-Key": string,
  "X-Google-API-Key": string,
  "X-Instance-Url": string
}
```

### 2.2 API de agente (`/api/agent`)

#### **Cuerpo de la solicitud:**
```typescript
{
  prompt: string,               // Prompt del usuario
  selectedModel: string,        // Nombre del agente IA
  data?: any                    // Datos adicionales
}
```

### 2.3 API de modelo (`/api/model`)

#### **Cuerpo de la solicitud:**
```typescript
{
  name: string                  // Nombre del modelo a descargar
}
```

## 3. Entradas de configuración

### 3.1 Parámetros de URL

#### **Configuración de instancia:**
- `instance_url`: URL de la instancia de Airflows
- `access_token`: Token de autenticación
- `disable_streaming`: Bandera booleana para transmisión
- `openai_api_key`: Clave API de OpenAI
- `google_api_key`: Clave API de Google AI

### 3.2 Variables de entorno

#### **Configuración del backend:**
- `OLLAMA_URL`: URL del servidor Ollama
- `AIRFLOWS_INSTANCE_URL`: URL de la plataforma Airflows
- `GOOGLE_API_KEY`: Clave API de Google AI
- `OPENAI_API_KEY`: Clave API de OpenAI

## 4. Entradas de GraphQL

### 4.1 Consultas de agente

#### **Consulta de metadatos de agente:**
```graphql
{
  agents: aia_AgentList(
    where: {
      type: {EQ: Assistant}
      name: {EQ: "${selectedModel}"}
    }
  ) {
    id
    name
    description
    model: ModelViaModel {
      id
      name
      provider: ProviderViaProvider {
        id
        name
      }
    }
    contexts: AgentContextListViaAgent {
      id
      name
      type
      function
      prompt
    }
    tools: AgentToolListViaAgent {
      tool: ToolViaTool {
        id
        name
        isPrivate: private
        description
        function
        categories
        parameters: ToolParameterListViaTool {
          id
          name
          description
          type
        }
      }
    }
  }
}
```

## 5. Entradas de ejecución de herramientas

### 5.1 Parámetros dinámicos de herramientas

#### **Generación de esquema de herramientas:**
- Parámetros cargados dinámicamente desde GraphQL
- Validación de esquema Zod
- Manejo de parámetros con seguridad de tipos
- Documentación de parámetros basada en descripción

#### **Ejecución de herramientas:**
```typescript
{
  functionName: string,         // Nombre de la función de herramienta
  parameters: object,           // Parámetros específicos de la herramienta
  headers: object,             // Encabezados de autorización
  body: object                 // Carga útil de la solicitud
}
```

## 6. Entradas de gestión de estado

### 6.1 Almacén de chat (Zustand)

#### **Datos de sesión de chat:**
```typescript
{
  chats: Record<string, ChatSession>,
  currentChatId: string | null,
  selectedModel: string | null,
  userName: string,
  base64Images: string[] | null,
  isDownloading: boolean,
  downloadProgress: number,
  downloadingModel: string | null
}
```

### 6.2 Contexto de aplicación

#### **Estado global:**
```typescript
{
  token: string | null,
  disableStreaming: boolean | null,
  instanceUrl: string | null,
  openaiAPIKey: string | null,
  googleAPIKey: string | null,
  isSpeakingEnabled: boolean
}
```

## 7. Entradas de interacción del usuario

### 7.1 Interacciones del ratón

#### **Clics de botón:**
- Botón de enviar mensaje
- Botón de detener generación
- Botón de entrada de voz
- Botón de carga de imagen
- Botón de configuración
- Botón de nuevo chat
- Botón de eliminar chat

#### **Selecciones de menú desplegable:**
- Selector de agente IA
- Menú de configuración
- Navegación del historial de chat

### 7.2 Interacciones del teclado

#### **Atajos:**
- `Enter`: Enviar mensaje
- `Shift + Enter`: Nueva línea
- `Escape`: Cerrar diálogos
- `Ctrl/Cmd + K`: Enfocar entrada de chat

## 8. Entradas de servicios externos

### 8.1 APIs de proveedores de IA

#### **API de OpenAI:**
- Autenticación con clave API
- Selección de modelo
- Respuestas de transmisión
- Llamada de herramientas

#### **API de Google AI:**
- Autenticación con clave API
- Acceso a modelos Gemini
- Llamada de funciones

#### **API de Ollama:**
- Gestión de modelos locales
- Descarga de modelos
- Solicitudes de inferencia

### 8.2 Plataforma Airflows

#### **Autenticación:**
- Inicio de sesión con usuario/contraseña
- Autenticación basada en tokens
- Gestión de sesiones

#### **Consultas GraphQL:**
- Recuperación de metadatos de agentes
- Configuración de herramientas
- Gestión de contexto

## 9. Validación y procesamiento de entradas

### 9.1 Validación del lado cliente

#### **Validación de formularios (Zod):**
```typescript
// Validación de nombre de usuario
const formSchema = z.object({
  username: z.string().min(2, {
    message: "El nombre debe tener al menos 2 caracteres.",
  }),
});

// Validación de nombre de modelo
const formSchema = z.object({
  name: z.string().min(1, {
    message: "Por favor selecciona un modelo para descargar",
  }),
});
```

### 9.2 Validación del lado servidor

#### **Validación de solicitudes API:**
- Verificación de campos requeridos
- Validación de tipos de datos
- Límites de tamaño de archivo
- Validación de formato de imagen

### 9.3 Manejo de errores

#### **Escenarios de error de entrada:**
- Claves API inválidas
- Problemas de conectividad de red
- Fallos de carga de archivos
- Errores de reconocimiento de voz
- Fallos de descarga de modelos
- Fallos de autenticación

## 10. Resumen de flujo de entradas

### 10.1 Flujo de entrada principal
```
1. Usuario ingresa texto/voz/imagen → 
2. Validación de entrada → 
3. Formación de solicitud API → 
4. Procesamiento del backend → 
5. Interacción con modelo IA → 
6. Generación de respuesta → 
7. Actualización de UI
```

### 10.2 Flujo de entrada de configuración
```
1. Parámetros de URL → 
2. Variables de entorno → 
3. Contexto de aplicación → 
4. Estado del componente → 
5. Encabezados de API
```

### 10.3 Flujo de ejecución de herramientas
```
1. Solicitud del usuario → 
2. Extracción de parámetros de herramienta → 
3. Recuperación de metadatos GraphQL → 
4. Generación de esquema de herramienta → 
5. Validación de parámetros → 
6. Ejecución de herramienta → 
7. Procesamiento de resultados
```

## 11. Consideraciones de seguridad de entradas

### 11.1 Autenticación
- Autenticación basada en tokens
- Gestión de claves API
- Validación de sesión

### 11.2 Validación de datos
- Sanitización de entradas
- Validación de tipo de archivo
- Aplicación de límites de tamaño

### 11.3 Privacidad
- Almacenamiento local para datos de usuario
- Transmisión segura de datos sensibles
- Sin exposición de claves API del lado cliente

