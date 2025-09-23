# Análisis memory leaks


- [Análisis memory leaks](#an%C3%A1lisis-memory-leaks)
    - [Lista resumida de memory leaks](#lista-resumida-de-memory-leaks)
    - [Explicación detallada de cada leak](#explicaci%C3%B3n-detallada-de-cada-leak)
    - [Problemas más críticos a resolver](#problemas-m%C3%A1s-cr%C3%ADticos-a-resolver)


## Lista resumida de memory leaks

### 1. **Event listeners no eliminados**
- `EntityList.js` línea 431: `window.addEventListener("scroll", ...)` sin cleanup
- `EntityView.js` línea 351: `window.addEventListener("message", ...)` sin cleanup

### 2. **Timeouts no limpiados**
- `AutoComplete.js` líneas 400-401, 429-430: `setTimeout` sin cleanup en `componentWillUnmount`
- `EntityList.js` y `EntityView.js`: Múltiples `setTimeout(this.refreshDataThrottled(), 500)` sin variables

### 3. **Funciones throttled sin cleanup**
- `EntityList.js` línea 420: `throttle(500, this.refreshData)` sin `.cancel()`
- `EntityView.js` línea 216: `throttle(500, this.refreshData)` sin `.cancel()`

### 4. **Refs acumulándose**
- `EntityList.js`: `this.basicFilterRefs = {}` sin limpieza
- `EntityView.js`: `this.inputRefs = {}` sin limpieza

### 5. **Variables globales**
- `EntityList.js` línea 95: `let idByName = {}` acumulando datos

### 6. **Librerías de terceros sin cleanup**
- React Big Calendar, Leaflet, BPMN.js, Blockly, Tesseract.js
- Service Worker sin deregistro

### 7. **Caché sin límites**
- Datos de GraphQL acumulándose
- Imágenes y documentos sin cleanup
- Opciones de autocompletado sin límite

### 8. **WebSockets y conexiones**
- Conexiones que permanecen abiertas
- Event listeners de WebSocket sin cleanup

### 9. **Manejo de archivos**
- File objects no liberados
- Blob URLs no revocadas
- FileReader sin abort()



## Explicación detallada de cada leak
### 1. **Memory leaks en event listeners**

**Ubicación**: Múltiples archivos
- **`EntityList.js`** (línea 431): Event listener de scroll de ventana agregado pero nunca removido
  ```javascript
  window.addEventListener("scroll", this.updateDimensions.bind(this));
  ```
  - El método `componentWillUnmount` no remueve este listener
  - Esto puede causar fugas de memoria cuando los componentes se desmontan y remontan

- **`EntityView.js`** (línea 351): Event listener de mensajes de ventana agregado pero nunca removido
  ```javascript
  window.addEventListener("message", this.handleMessage);
  ```
  - No hay limpieza en `componentWillUnmount`

- **`SignaturePad.js`** (líneas 42, 91): Correctamente gestionado con add/remove
  ```javascript
  window.addEventListener('resize', this._callResizeHandler);
  // Correctamente removido en componentWillUnmount
  window.removeEventListener('resize', this._callResizeHandler);
  ```

### 2. **Memory leaks en timeouts e intervals**

**Ubicación**: Múltiples archivos
- **`AutoComplete.js`** (líneas 400-401, 429-430): Timeouts que pueden no ser limpiados correctamente
  ```javascript
  clearTimeout(this.timeoutId);
  this.timeoutId = setTimeout(() => {
    this.refreshData(inputValue, resolve);
  }, 200);
  ```
  - Aunque los timeouts se limpian antes de establecer nuevos, no hay limpieza en `componentWillUnmount`

- **`EntityList.js`** y **`EntityView.js`**: Múltiples llamadas `setTimeout` con funciones throttled
  ```javascript
  setTimeout(this.refreshDataThrottled(), 500);
  ```
  - Estos timeouts no se almacenan en variables y no pueden ser limpiados

### 3. **Memory leaks en funciones throttled**

**Ubicación**: `EntityList.js` y `EntityView.js`
- **`EntityList.js`** (línea 420): Funciones throttled creadas pero nunca limpiadas
  ```javascript
  this.refreshDataThrottled = throttle(500, this.refreshData);
  this.handleDeleteDebounced = debounce(3000, true, this.handleDelete);
  ```
- **`EntityView.js`** (línea 216): Problema similar
  ```javascript
  this.refreshDataThrottled = throttle(500, this.refreshData);
  ```

### 4. **Problemas en el ciclo de vida de componentes React**

**Ubicación**: Múltiples archivos
- **`EntityList.js`** y **`EntityView.js`**: Patrón de bandera `_isMounted`
  ```javascript
  this._isMounted = false; // en componentWillUnmount
  ```
  - Aunque esto previene actualizaciones de estado después del desmontaje, no limpia recursos

### 5. **Acumulación de estado de componentes grandes**

**Ubicación**: `EntityList.js` y `EntityView.js`
- **`EntityList.js`**: Objetos de estado grandes que crecen con datos
  ```javascript
  this.state = {
    data: [],
    attributes: [],
    // ... muchas otras propiedades
  };
  ```
- **`EntityView.js`**: Objetos de estado grandes similares
  ```javascript
  this.state = {
    data: null,
    uploadedData: {},
    // ... muchas otras propiedades
  };
  ```

### 6. **Acumulación de refs**

**Ubicación**: Múltiples archivos
- **`EntityList.js`**: Múltiples refs que pueden acumularse
  ```javascript
  this.basicFilterRefs = {};
  this.searchInput = React.createRef();
  ```
- **`EntityView.js`**: Acumulación de refs similar
  ```javascript
  this.inputRefs = {};
  this.uploadMrz = React.createRef();
  ```

### 7. **Memory leaks en librerías de terceros**

**Ubicación**: Múltiples archivos
- **`EntityList.js`**: React Big Calendar, Leaflet, Diagram libraries
  ```javascript
  import { Calendar, momentLocalizer } from 'react-big-calendar';
  import { Map, Marker, TileLayer } from 'react-leaflet';
  import createEngine from '@projectstorm/react-diagrams';
  ```
  - Estas librerías pueden tener sus propios problemas de gestión de memoria

- **`EntityView.js`**: BPMN.js, Blockly, Tesseract.js
  ```javascript
  import Modeler from 'bpmn-js/lib/Modeler';
  import { Block, Value, Field, Shadow } from './blockly';
  import Tesseract from 'tesseract.js';
  ```

### 8. **Acumulación de variables globales**

**Ubicación**: `EntityList.js` (línea 95)
- **`idByName`**: Objeto global que acumula datos
  ```javascript
  let idByName = {};
  ```

### 9. **Memory leaks en service worker**

**Ubicación**: `serviceWorker.js`
- El registro del service worker y event listeners pueden no ser limpiados correctamente

### 10. **Memory leaks en context provider**

**Ubicación**: `App.js`
- **`AppContext.Provider`**: Objeto de contexto grande que puede acumular datos
  ```javascript
  export const AppContext = React.createContext();
  ```

### 11. **Problemas de rendimiento en renderizado**

**Ubicación**: Múltiples archivos
- **Renderizado innecesario**: Componentes que se re-renderizan sin cambios reales
- **Funciones inline**: Funciones creadas en cada render que causan re-renderizados
- **Objetos inline**: Objetos creados en cada render que causan re-renderizados

### 12. **Acumulación de datos en caché**

**Ubicación**: Múltiples archivos
- **Caché de consultas**: Datos de GraphQL que se acumulan sin límite
- **Caché de imágenes**: Imágenes y documentos que no se limpian
- **Caché de autocompletado**: Opciones de autocompletado que se acumulan

### 13. **Problemas con WebSockets y conexiones**

**Ubicación**: Múltiples archivos
- **Conexiones no cerradas**: Conexiones que permanecen abiertas
- **Event listeners de WebSocket**: Listeners que no se limpian correctamente

### 14. **Fugas en manejo de archivos**

**Ubicación**: `EntityView.js`
- **File objects**: Objetos de archivo que no se liberan correctamente
- **Blob URLs**: URLs de blob que no se revocan
- **FileReader**: Instancias de FileReader que no se limpian

### 15. **Problemas con librerías de UI**

**Ubicación**: Múltiples archivos
- **Material-UI**: Componentes que pueden acumular listeners internos
- **React Select**: Instancias que no se limpian correctamente
- **React Big Calendar**: Eventos que no se limpian

## Problemas más críticos a resolver

1. **Event listeners en `EntityList.js` y `EntityView.js`** que nunca se remueven
2. **Funciones throttled** que no se limpian correctamente
3. **Timeouts** que no se almacenan en variables y no pueden ser limpiados
4. **Acumulación de datos** en caché sin límites
5. **Librerías de terceros** sin cleanup adecuado

Estos problemas pueden causar fugas de memoria significativas con el tiempo, especialmente en aplicaciones con montaje/desmontaje frecuente de componentes.