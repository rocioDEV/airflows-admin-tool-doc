# estructura y convenciones de docs

Este documento explica cómo está organizado el directorio `docs/` y dónde colocar nuevo contenido. También define convenciones de nombres, idioma y categorías/sidebar.

---

## estructura general

```
/docs
├── admin-tool/                # análisis y planes del Admin Tool legado
│   ├── migration-status/      # progreso de migración y regresiones
│   ├── old-app-analysis/      # análisis en profundidad de la app antigua
│   └── proposals/             # propuestas y RFCs
├── airflows2_0/               # documentación de producto Airflows 2.0
│   ├── core/                  # conceptos core, arquitectura, how-tos
│   ├── workflows/             # modelo de workflows, guías, APIs
│   ├── ia/                    # IA/agentes, automatización en Airflows 2.0
│   ├── technical_docs/        # docs técnicas legacy (a migrar a lo anterior)
│   └── img/                   # imágenes compartidas de esta sección
├── workflows/                 # docs conceptuales/API de workflows (legacy)
└── *.md                       # visiones generales o notas transversales
```

---

## dónde colocar las cosas

- **Core de Airflows 2.0 (arquitectura y how-tos)**: en `docs/airflows2_0/core/`.
  - Ejemplo (legacy): la guía en español para añadir un endpoint HTTP está en `airflows2_0/technical_docs/crear_un_nuevo_endpoint.md`. Las nuevas guías de este tipo deben ir en `airflows2_0/core/`.
- **Workflows (Airflows 2.0)**: en `docs/airflows2_0/workflows/`.
- **IA / agentes / automatización**: en `docs/airflows2_0/ia/`.
- **Migración/análisis del Admin Tool**: en `docs/admin-tool/`.
  - Estado en curso → `admin-tool/migration-status/`
  - Análisis de la app antigua → `admin-tool/old-app-analysis/`
  - Propuestas de diseño → `admin-tool/proposals/`
- **Temas generales o transversales** (no ligados a una sección): en la raíz de `docs/`.
- **Imágenes**: preferir un directorio `img/` junto al documento o en la raíz de la sección; referenciar con rutas relativas.

---

## nombres y ordenación

- **Títulos**: usar sentence case (solo la primera palabra en mayúscula).
- **Nombres de fichero**:
  - Preferir minúsculas con guiones o subrayados (p.ej., `crear_un_nuevo_endpoint.md` o `crear-un-nuevo-endpoint.md`). Mantener estilos existentes al editar.
  - Usar prefijos numéricos para forzar orden cuando sea necesario (p.ej., `1intro.md`, `2setup.md`).
- **Un único h1 por doc**: comenzar con un solo `#` de título.

---

## idioma

- El idioma por defecto es español. El inglés es bienvenido cuando mejore la claridad para la audiencia.
- Si añades la contraparte en inglés, colócala cerca del original (misma carpeta) y deja clara la pareja:
  - Opción A: crear una carpeta hermana `en/` y poner ahí el fichero en inglés.
  - Opción B: sufijar el nombre del fichero con `.en.md`.
- Enlaza ambas versiones al inicio de cada documento.

---

## sidebar y categorías (docusaurus)

Usa `_category_.json` en carpetas que deban aparecer agrupadas en el sidebar. Ejemplo típico:

```json
{
  "label": "Technical docs",
  "position": 2,
  "link": { "type": "generated-index", "title": "Technical docs" }
}
```

- **label**: nombre de la sección que se muestra en el sidebar.
- **position**: orden entre hermanas (más bajo aparece antes).
- **link**: usa `generated-index` salvo que mantengas un índice manual.

Coloca `_category_.json` en directorios como `airflows2_0/core/`, `airflows2_0/workflows/`, `airflows2_0/ia/`, etc., para controlar anidación y orden.

---

## checklist de autoría

- **Ubicación**: elige la carpeta usando las reglas anteriores.
- **Título**: sentence case; un único `#` h1.
- **Imágenes**: guarda en un `img/` hermano y usa rutas relativas.
- **Enlaces**: usa enlaces relativos; incluye rutas de ficheros en monospace cuando referencies código (p.ej., `com.niledb.platform/src/main/java/verticles/HttpVerticle.java`).
- **Sin duplicados**: al mover contenido, elimina el fichero antiguo y actualiza enlaces.

---

## ejemplos

- Guía de nuevo endpoint (español): `docs/airflows2_0/technical_docs/crear_un_nuevo_endpoint.md` (legacy; nuevas → `airflows2_0/core/`)
- Visión del modelo de workflows: `docs/airflows2_0/workflows_model.md` (a migrar a `airflows2_0/workflows/`)
- Estado de migración del Admin Tool: `docs/admin-tool/migration-status/`
