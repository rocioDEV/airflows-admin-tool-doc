---
sidebar_position: 1
---


# Tipos de campos

## Descripción general

El componente `EntityField` es responsable de renderizar diferentes tipos de campos de formulario basándose en las propiedades y configuración del atributo. El componente utiliza una serie de verificaciones condicionales para determinar qué componente de campo específico renderizar para cada atributo.

## Conceptos principales

### Lógica de determinación de tipo de campo

El componente `EntityField` sigue un orden específico de verificaciones para determinar qué tipo de campo renderizar. El orden es importante ya que asegura que los tipos de campos más específicos se manejen primero.

### Verificaciones principales de tipo de campo

El componente verifica los tipos de campos en el siguiente orden:

1. **SIGNATURE** - Campos de firma manuscrita
2. **DOCUMENT** - Campos de carga de archivos/documentos
3. **BARCODE** - Campos de escaneo de códigos de barras
4. **Campos de acción** - Campos con propiedades de acción
5. **Campos de texto básicos** - Campos de entrada de texto simples
6. **Campos de contraseña** - Entrada segura de contraseña
7. **Selector de color** - Campos de selector de color
8. **Texto enriquecido** - Campos de editor de texto enriquecido
9. **Selector de icono** - Campos de selector de icono
10. **Campos enum** - Campos de dropdown/select
11. **Campos booleanos** - Campos de checkbox/toggle
12. **Campos numéricos** - Campos de entrada numérica
13. **Campos de fecha** - Campos de selector de fecha
14. **Campos de timestamp** - Campos de selector de fecha y hora
15. **Campos de hora** - Campos de selector de hora
16. **Campos de texto array** - Campos de múltiples valores de texto
17. **Campos POINT** - Campos de coordenadas geográficas

## Cómo funciona el sistema de tipos de campos

### Determinación del tipo de campo

El componente determina el tipo de campo usando la variable `fieldType`:

```typescript
const fieldType = entityLocalModel.attributes?.[attribute.name]?.type ?? undefined
```

Este valor viene de la configuración del modelo local y determina qué componente de campo específico renderizar.

### Lógica de estado deshabilitado

Todos los campos comparten una lógica común de estado deshabilitado:

```typescript
const disabled = 
  (!isEntityEmpty && attribute.editableOnInsertOnly) ||
  attribute.computed ||
  mode === 'view' ||
  (!model?.super &&
    ((!isEntityEmpty && attribute.privileges?.['UPDATE'] === undefined) ||
     (isEntityEmpty && attribute.privileges?.['INSERT'] === undefined)))
```

## Definiciones de tipos

### Interfaz Attribute

```typescript
export type Attribute = {
  name: string
  type: string
  array: boolean
  entityName: string
  enumType: {
    schema: string
    name: string
    values: Record<string, string>
  }
  textFilter: string
  computed: undefined
  privileges?: {
    SELECT?: string
    INSERT?: string
    UPDATE?: string
    DELETE?: string
    MENU?: string
  }
  xs: GridSize
  sm: GridSize
  spaceReserved: boolean
  editableOnInsertOnly: boolean
  variantSelector: string
  // Custom per type
  length: number
  pattern: string
  _autoFocus: boolean
  barcodeType: string
  rich: boolean
  password: boolean
  color: boolean
  icon: boolean
  multiline: boolean
  required: boolean
  richTextEditorModules: string
  acceptedFileTypes: string[]
  maxFiles: number
  browseZip: boolean
  hidden: boolean
  min: number
  max: number
  step: number
  referenceAttributeName: string
  externalSelectorUrl: string
  incomingReferenceAttributeName: string
  referencedKey: {
    entityName: string
    attributeName: string
  }
  externalSelectorTarget: string
  externalSelectorWindowFeatures: string
  addAccessToken: boolean
  externalSelectorEditableValue: boolean
  variants: string[]
  disableThousandsSeparator: boolean
  prefix: string
  suffix: string
  order: number
  label: string
  labelLanguage: string
  orderInList: number
  tab: string
}
```

### Interfaz EntityFieldState

```typescript
export type EntityFieldState = {
  message: string
  messageError: boolean
  messageOpened: boolean
  data: Record<string, string | string[] | boolean | null>
  uploadedData?: UploadedFileData[]
  variantAttributes: VariantAttribute[]
  variants: string[]
  contents: string
}
```

## Descripciones detalladas de tipos de campos

### 1. Campos de firma

**Condición**: `fieldType === 'SIGNATURE'`

```typescript
if (fieldType === 'SIGNATURE') {
  return (
    <SignatureField
      attribute={attribute}
      entity={entity}
      entityLocalModel={entityLocalModel}
      mode={mode}
      formControl={formControl}
      disabled={disabled}
    />
  )
}
```

**Componente**: `SignatureField` de `src/components/entity-view/entity-field/SignatureField/SingatureField.tsx`
**Propósito**: Permite a los usuarios capturar firmas manuscritas
**Propiedades**: Usa funcionalidad de pad de firma

### 2. Campos de documento

**Condición**: `!attribute.array && fieldType === 'DOCUMENT'`

```typescript
if (!attribute.array && fieldType === 'DOCUMENT') {
  return (
    <DocumentField
      disabled={isDisabled}
      acceptedFiles={isEmpty(attribute.acceptedFileTypes) ? [] : attribute.acceptedFileTypes}
      filesLimit={attribute.array ? (isEmpty(attribute.maxFiles) ? 5 : attribute.maxFiles) : 1}
      maxFileSize={documentMaxFileSizeMB ? documentMaxFileSizeMB * 1024 * 1024 : undefined}
      entity={entity}
      entityId={entityId}
      attribute={attribute}
      formControl={formControl}
      setFieldState={setFieldState}
      fieldState={fieldState}
    />
  )
}
```

**Componente**: `DocumentField` de `src/components/entity-view/entity-field/DocumentField/DocumentField.tsx`
**Propósito**: Maneja cargas de archivos y gestión de documentos
**Propiedades**:
- `acceptedFileTypes`: Array de tipos de archivo permitidos
- `maxFiles`: Número máximo de archivos (por defecto: 5 para arrays, 1 para único)
- `maxFileSize`: Tamaño máximo de archivo en bytes

### 3. Campos de código de barras

**Condición**: `fieldType === 'BARCODE'`

```typescript
if (fieldType === 'BARCODE') {
  return (
    <BarcodeField
      attribute={attribute}
      isEntityEmpty={isEntityEmpty}
      formControl={formControl}
      mode={mode}
      model={model}
    />
  )
}
```

**Componente**: `BarcodeField` de `src/components/entity-view/entity-field/BarcodeField.tsx`
**Propósito**: Maneja escaneo e entrada de códigos de barras
**Propiedades**: Usa funcionalidad de escaneo de códigos de barras

### 4. Campos de acción

**Condición**: `isActionAttribute(entityLocalModel, attribute)`

```typescript
if (isActionAttribute(entityLocalModel, attribute)) {
  return (
    <ActionField
      fieldState={fieldState}
      attribute={attribute}
      entity={entity}
      formControl={formControl}
    />
  )
}
```

**Componente**: `ActionField` de `src/components/entity-view/entity-field/ActionField.tsx`
**Propósito**: Campos que activan acciones o flujos de trabajo
**Propiedades**:
- `action`: Configuración de acción
- `actionVisibleInForm`: Si la acción es visible en el formulario

### 5. Campos de texto básicos

**Condición**: Verificación condicional compleja para atributos de texto simples

```typescript
if (
  !attribute.array &&
  !isActionAttribute(entityLocalModel, attribute) &&
  !attribute.rich &&
  !attribute.password &&
  !attribute.color &&
  !attribute.icon &&
  (isTextAttribute(attribute) || attribute.type === 'VECTOR') &&
  attribute.enumType === undefined
) {
  return (
    <TextField
      attribute={attribute}
      mode={mode}
      entity={entity}
      entityLocalModel={entityLocalModel}
      fieldType={fieldType}
      formControl={formControl}
      disabled={disabled}
    />
  )
}
```

**Componente**: `TextField` de `src/components/entity-view/entity-field/TextField.tsx`
**Propósito**: Campos de entrada de texto estándar
**Propiedades**:
- `type`: 'TEXT', 'CHAR', 'VARCHAR', o 'VECTOR'
- `multiline`: Si el campo soporta múltiples líneas
- `_autoFocus`: Si el campo debe auto-enfocarse

### 6. Campos de contraseña

**Condición**: `!attribute.array && !attribute.rich && attribute.password && isTextAttribute(attribute) && attribute.enumType === undefined`

```typescript
const isPassword =
  !attribute.array &&
  !attribute.rich &&
  attribute.password &&
  isTextAttribute(attribute) &&
  attribute.enumType === undefined

if (isPassword) {
  return (
    <TextField
      attribute={attribute}
      mode={mode}
      entity={entity}
      entityLocalModel={entityLocalModel}
      fieldType={fieldType}
      formControl={formControl}
      autoComplete="new-password"
      type="password"
      disabled={disabled}
    />
  )
}
```

**Componente**: `TextField` con tipo password
**Propósito**: Entrada segura de contraseña
**Propiedades**:
- `type`: "password"
- `autoComplete`: "new-password"

### 7. Campos selector de color

**Condición**: `!attribute.array && attribute.color && isTextAttribute(attribute)`

```typescript
if (!attribute.array && attribute.color && isTextAttribute(attribute)) {
  return (
    <ColorSelectorField
      attribute={attribute}
      entityLocalModel={entityLocalModel}
      entity={entity}
      formControl={formControl}
      disabled={disabled}
    />
  )
}
```

**Componente**: `ColorSelectorField` de `src/components/entity-view/entity-field/ColorSelectorField.tsx`
**Propósito**: Interfaz de selector de color
**Propiedades**: `color: true`

### 8. Campos de texto enriquecido

**Condición**: `!attribute.array && !isActionAttribute(entityLocalModel, attribute) && attribute.rich && !attribute.password && !attribute.color && !attribute.icon && isTextAttribute(attribute) && attribute.enumType === undefined`

```typescript
if (
  !attribute.array &&
  !isActionAttribute(entityLocalModel, attribute) &&
  attribute.rich &&
  !attribute.password &&
  !attribute.color &&
  !attribute.icon &&
  isTextAttribute(attribute) &&
  attribute.enumType === undefined
) {
  return (
    <RichTextField
      attribute={attribute}
      disabled={disabled}
      fieldType={fieldType}
      entityLocalModel={entityLocalModel}
      entity={entity}
      mode={mode}
      formControl={formControl}
    />
  )
}
```

**Componente**: `RichTextField` de `src/components/entity-view/entity-field/RichTextField.tsx`
**Propósito**: Editor de texto enriquecido con opciones de formato
**Propiedades**:
- `rich: true`
- `richTextEditorModules`: Configuración para el editor de texto enriquecido

### 9. Campos selector de icono

**Condición**: `isTextAttribute(attribute) && attribute.enumType && attribute.icon`

```typescript
if (isTextAttribute(attribute) && attribute.enumType && attribute.icon) {
  return (
    <IconField
      attribute={attribute}
      entity={entity}
      entityLocalModel={entityLocalModel}
      isDisabled={isDisabled}
      formControl={formControl}
    />
  )
}
```

**Componente**: `IconField` de `src/components/entity-view/entity-field/IconField.tsx`
**Propósito**: Interfaz de selector de icono
**Propiedades**:
- `icon: true`
- `enumType`: Configuración de enum para iconos disponibles

### 10. Campos enum

**Condición**: `isTextAttribute(attribute) && attribute.enumType !== undefined`

```typescript
if (isTextAttribute(attribute) && attribute.enumType !== undefined) {
  return (
    <EnumField
      attribute={attribute}
      entityLocalModel={entityLocalModel}
      entity={entity}
      formControl={formControl}
      setFieldState={setFieldState}
      fieldState={fieldState}
      isDisabled={disabled}
    />
  )
}
```

**Componente**: `EnumField` de `src/components/entity-view/entity-field/EnumField/EnumField.tsx`
**Propósito**: Campos de dropdown/select con opciones predefinidas
**Propiedades**:
- `enumType`: Configuración de enum
- `array`: Si se permiten múltiples valores

### 11. Campos booleanos

**Condición**: `!attribute.array && attribute.type === 'BOOLEAN'`

```typescript
if (!attribute.array && attribute.type === 'BOOLEAN') {
  return (
    <BooleanField
      attribute={attribute}
      entity={entity}
      mode={mode}
      model={model}
      isEntityEmpty={isEntityEmpty}
      formControl={formControl}
    />
  )
}
```

**Componente**: `BooleanField` de `src/components/entity-view/entity-field/BooleanField.tsx`
**Propósito**: Campos de checkbox/toggle
**Propiedades**: `type: 'BOOLEAN'`

### 12. Campos numéricos

**Condición**: `!attribute.array && isNumberAttribute(attribute)`

```typescript
if (!attribute.array && isNumberAttribute(attribute)) {
  return (
    <TextField
      type="number"
      attribute={attribute}
      mode={mode}
      entity={entity}
      entityLocalModel={entityLocalModel}
      fieldType={fieldType}
      formControl={formControl}
      disabled={disabled}
    />
  )
}
```

**Componente**: `TextField` con tipo number
**Propósito**: Campos de entrada numérica
**Tipos soportados**:
- `INTEGER`, `SMALLINT`, `BIGINT`
- `SERIAL`, `SMALLSERIAL`, `BIGSERIAL`
- `DECIMAL`, `DOUBLE_PRECISION`, `REAL`
- `MONEY`

### 13. Campos de fecha

**Condición**: `!attribute.array && attribute.type === 'DATE'`

```typescript
if (!attribute.array && attribute.type === 'DATE') {
  return (
    <TextField
      attribute={attribute}
      mode={mode}
      entity={entity}
      entityLocalModel={entityLocalModel}
      fieldType={fieldType}
      formControl={formControl}
      type="date"
      disabled={disabled}
    />
  )
}
```

**Componente**: `TextField` con tipo date
**Propósito**: Campos de selector de fecha
**Propiedades**: `type: 'DATE'`

### 14. Campos de timestamp

**Condición**: `!attribute.array && (attribute.type === 'TIMESTAMP' || attribute.type === 'TIMESTAMP_WITH_TIME_ZONE')`

```typescript
if (
  !attribute.array &&
  (attribute.type === 'TIMESTAMP' || attribute.type === 'TIMESTAMP_WITH_TIME_ZONE')
) {
  return (
    <TextField
      attribute={attribute}
      mode={mode}
      entity={entity}
      entityLocalModel={entityLocalModel}
      fieldType={fieldType}
      formControl={formControl}
      type="datetime-local"
      disabled={disabled}
    />
  )
}
```

**Componente**: `TextField` con tipo datetime-local
**Propósito**: Campos de selector de fecha y hora
**Propiedades**: `type: 'TIMESTAMP'` o `'TIMESTAMP_WITH_TIME_ZONE'`

### 15. Campos de hora

**Condición**: `!attribute.array && (attribute.type === 'TIME' || attribute.type === 'TIME_WITH_TIME_ZONE')`

```typescript
if (!attribute.array && (attribute.type === 'TIME' || attribute.type === 'TIME_WITH_TIME_ZONE')) {
  return (
    <TextField
      attribute={attribute}
      mode={mode}
      entity={entity}
      entityLocalModel={entityLocalModel}
      fieldType={fieldType}
      formControl={formControl}
      type="time"
      disabled={disabled}
    />
  )
}
```

**Componente**: `TextField` con tipo time
**Propósito**: Campos de selector de hora
**Propiedades**: `type: 'TIME'` o `'TIME_WITH_TIME_ZONE'`

### 16. Campos de texto array

**Condición**: `attribute.array && isTextAttribute(attribute) && attribute.enumType === undefined`

```typescript
if (attribute.array && isTextAttribute(attribute) && attribute.enumType === undefined) {
  return (
    <TextMultipleField
      attribute={attribute}
      entity={entity}
      disabled={disabled}
      formControl={formControl}
    />
  )
}
```

**Componente**: `TextMultipleField` de `src/components/entity-view/entity-field/TextMultipleField.tsx`
**Propósito**: Campos de múltiples valores de texto
**Propiedades**:
- `array: true`
- Tipo de atributo de texto (TEXT, CHAR, VARCHAR)

### 17. Campos POINT

**Condición**: `!attribute.array && attribute.type === 'POINT'`

```typescript
if (!attribute.array && attribute.type === 'POINT') {
  return (
    <PointField
      attribute={attribute}
      entityLocalModel={entityLocalModel}
      isEntityEmpty={isEntityEmpty}
      mode={mode}
      model={model}
      entity={entity}
      formControl={formControl}
      registerField={registerField}
    />
  )
}
```

**Componente**: `PointField` de `src/components/entity-view/entity-field/PointField.tsx`
**Propósito**: Campos de coordenadas geográficas (latitud/longitud)
**Propiedades**: `type: 'POINT'`

## Integración con otros sistemas

### Campos externos

Los campos externos se manejan por separado en el componente `EntityViewFieldGroup`:

```typescript
const isExternalSearch = attribute.externalSelectorUrl && attributeTab === tab && attributeGroup === group

if (isExternalSearch) {
  return <ExternalField />
}
```

**Componente**: `ExternalField` de `src/components/entity-view/entity-field/ExternalField.tsx`
**Propósito**: Campos que abren selectores externos o interfaces de búsqueda
**Propiedades**:
- `externalSelectorUrl`: URL para el selector externo
- `externalSelectorTarget`: Ventana objetivo para el selector externo
- `externalSelectorWindowFeatures`: Características de ventana para el selector externo

### Campos de referencia directa

Los campos de referencia directa también se manejan por separado:

```typescript
const isDirectReference = entityState.data && attribute.referenceAttributeName !== undefined && 
  (!attribute.externalSelectorUrl || attribute.externalSelectorUrl === '') && 
  referenceAttribute && 
  (!referenceAttribute.tab || referenceAttribute.tab === tab) && 
  (!referenceAttribute.group || referenceAttribute.group === group)

if (isDirectReference) {
  return <DirectReferenceField />
}
```

**Componente**: `DirectReferenceField` de `src/components/entity-view/entity-field/DirectReferenceField.tsx`
**Propósito**: Campos que referencian otras entidades directamente
**Propiedades**:
- `referenceAttributeName`: Nombre del atributo de referencia
- `referencedKey`: Información sobre la entidad referenciada

## Funciones relacionadas

### isTextAttribute

Ubicada en `src/components/entity-view/entity-field/EntityField.tsx` (líneas 381-383):

```typescript
function isTextAttribute(attribute: DBAttributeDefinition) {
  return attribute.type === 'TEXT' || attribute.type === 'CHAR' || attribute.type === 'VARCHAR'
}
```

**Propósito**: Determina si un atributo es de tipo texto
**Retorna**: `boolean`

### isNumberAttribute

Ubicada en `src/components/entity-view/util/isNumberAttribute.ts`:

```typescript
export function isNumberAttributeType(type?: string | null) {
  return (
    type === 'INTEGER' ||
    type === 'SMALLINT' ||
    type === 'BIGINT' ||
    type === 'SERIAL' ||
    type === 'DECIMAL' ||
    type === 'DOUBLE_PRECISION' ||
    type === 'REAL' ||
    type === 'MONEY' ||
    type === 'SMALLSERIAL' ||
    type === 'BIGSERIAL'
  )
}

export function isNumberAttribute(attribute?: Attribute | null) {
  return isNumberAttributeType(attribute?.type)
}
```

**Propósito**: Determina si un atributo es de tipo numérico
**Retorna**: `boolean`

### isActionAttribute

Ubicada en `src/components/entity-view/util/isActionAttribute.ts`:

```typescript
export function isActionAttribute(entityLocalModel: EntityConfig, attribute: DBAttributeDefinition) {
  return (
    entityLocalModel.attributes?.[attribute.name]?.action ||
    entityLocalModel.attributes?.[attribute.name]?.actionVisibleInForm
  )
}
```

**Propósito**: Determina si un atributo tiene propiedades de acción
**Retorna**: `boolean`

## Comportamiento de UI

### Gestión del estado del campo

Cada campo mantiene su propio estado a través de la interfaz `EntityFieldState`:

```typescript
export type EntityFieldState = {
  message: string
  messageError: boolean
  messageOpened: boolean
  data: Record<string, string | string[] | boolean | null>
  uploadedData?: UploadedFileData[]
  variantAttributes: VariantAttribute[]
  variants: string[]
  contents: string
}
```

### Lógica de estado deshabilitado

Los campos pueden deshabilitarse basándose en varias condiciones:

```typescript
const disabled = 
  (!isEntityEmpty && attribute.editableOnInsertOnly) ||
  attribute.computed ||
  mode === 'view' ||
  (!model?.super &&
    ((!isEntityEmpty && attribute.privileges?.['UPDATE'] === undefined) ||
     (isEntityEmpty && attribute.privileges?.['INSERT'] === undefined)))
```

**Condiciones**:
- `editableOnInsertOnly`: Campo solo editable durante la inserción
- `computed`: Campo es computado/calculado
- `mode === 'view'`: Formulario en modo vista
- Restricciones de privilegios: Usuario carece de privilegios UPDATE/INSERT

## Manejo de errores

### Campos no soportados

Si ningún tipo de campo coincide con la configuración del atributo, el componente renderiza un mensaje "NOT_SUPPORTED_YET":

```typescript
return (
  <>
    <Title>NOT_SUPPORTED_YET</Title>
    <span>
      {attribute.name + ', ' + attribute.type + ', ' + 
       (isEmpty(fieldState.data) ? '' : fieldState.data[attribute.name])}
    </span>
  </>
)
```

### Validación de tipo de campo

- Los tipos de campo inválidos se capturan y muestran con información de error
- Los componentes de campo faltantes resultan en el mensaje de campo no soportado
- Las incompatibilidades de tipo entre atributo y fieldType se manejan graciosamente

## Consideraciones de rendimiento

### Determinación del tipo de campo

- Las verificaciones de tipo de campo ocurren en cada renderizado
- Considera memorizar la determinación del tipo de campo para formularios complejos
- El orden de verificaciones está optimizado para los tipos de campo más comunes primero

### Renderizado de componentes

- Cada tipo de campo tiene su propio componente optimizado
- Los tipos de campo pesados (como editores de texto enriquecido) solo se renderizan cuando es necesario
- El estado del campo se maneja localmente para prevenir re-renderizados innecesarios

## Mejores prácticas

### Diseño de tipo de campo

- Usa el tipo de campo más específico para tus datos
- Considera la experiencia del usuario al elegir tipos de campo
- Asegúrate de que los tipos de campo coincidan con el modelo de datos subyacente

### Optimización de rendimiento

- Evita verificaciones innecesarias de tipo de campo
- Usa tipos de campo apropiados para los datos que se están recopilando
- Considera carga diferida para tipos de campo complejos

### Experiencia de usuario

- Proporciona etiquetas claras y texto de ayuda para cada tipo de campo
- Asegura comportamiento consistente a través de tipos de campo similares
- Maneja casos extremos graciosamente con mensajes de error apropiados 