# Campo de texto enriquecido

`RichTextField` es el editor de texto enriquecido que se renderiza cuando el atributo cumple estas condiciones:

- El atributo es de tipo texto
- Está marcado el parámetro "Es texto enriquecido"


El valor que se guarda en base de datos al editar un atributo de texto enriquecido es un string HTML.

## configuración: `richTextEditorModules`

`richTextEditorModules` acepta un JSON (como cadena de texto) con la configuración de módulos de React Quill (https://quilljs.com/docs/modules/toolbar). Si está vacío o `null`, se aplican los valores por defecto de Airflows (ver más abajo).


### valores por defecto (Airflows)

Si no se provee configuración, se usa esta barra de herramientas por defecto:

```json
{
  "toolbar": [
    [{ "font": [] }, { "size": ["small", false, "large", "huge"] }],
    ["bold", "italic", "underline", "strike"],
    ["blockquote", "code-block"],
    ["link"],
    [{ "color": [] }, { "background": [] }],
    [{ "header": [2, 3, 4, 5, 6, false] }],
    [{ "list": "ordered" }, { "list": "bullet" }, { "indent": "-1" }, { "indent": "+1" }],
    [{ "script": "sub" }, { "script": "super" }],
    [{ "header": 1 }, { "header": 2 }],
    [{ "indent": "-1" }, { "indent": "+1" }],
    [{ "align": [] }],
    ["clean"]
  ]
}
```

## ejemplos de configuración

Coloca una cadena JSON en `richTextEditorModules` dentro del modelo local del atributo. Ejemplos:

### 1) barra mínima (negrita, cursiva, enlaces)

```json
{
  "toolbar": [["bold", "italic", "underline"], ["link"]]
}
```

### 2) habilitar imágenes y vídeos

```json
{
  "toolbar": [
    ["bold", "italic", "underline"],
    ["link", "image", "video"],
    [{ "header": [1, 2, 3, false] }]
  ]
}
```


### 3) tamaños y encabezados personalizados

```json
{
  "toolbar": [
    [{ "size": ["small", false, "large", "huge"] }],
    [{ "header": [1, 2, 3, 4, 5, 6, false] }]
  ]
}
```

### 4) colores con lista por defecto del tema

Si defines `"color": []` o `"background": []`, el tema Snow de Quill usa su lista por defecto de colores.

```json
{
  "toolbar": [[{ "color": [] }, { "background": [] }]]
}
```

### 5) paleta de colores corporativos (texto y fondo)

```json
{
  "toolbar": [
    [
      { "color": ["#000000", "#1F2937", "#2563EB", "#DC2626", "#059669", "#F59E0B", false] },
      { "background": ["#FFFFFF", "#F3F4F6", "#DBEAFE", "#FEE2E2", "#D1FAE5", "#FEF3C7", false] }
    ],
    ["bold", "italic", "underline"],
    ["clean"]
  ]
}
```

Nota: incluye `false` para permitir “sin color/sin fondo” (restablecer al estilo por defecto).

### 6) solo selector de color de fondo (paleta reducida)

```json
{
  "toolbar": [
    [{ "background": ["#FFFFFF", "#FFF7ED", "#FEF3C7", "#ECFDF5", "#EEF2FF", false] }],
    [{ "align": [] }],
    ["clean"]
  ]
}
```
