---
sidebar_position: 1
---

# `localModel`


El model de Airflows(Entidades, Usuarios, Funciones, Paquetes web...) está almacenado de forma estática (un archivo JSON) y se empaqueta junto con el resto del frontal.

1. Entidades principales del negocio:

- `Models.Entity`: La entidad central que representa objetos/tablas de negocio. Tiene atributos como esquema, nombre, idioma, y puede configurarse con menús, iconos y documentación.
- `Models.EntityAttribute`: Define los atributos/campos de las entidades, con varios tipos y configuraciones para visualización en UI, validación y comportamiento.
- `Models.EntityKey`: Representa claves primarias y restricciones únicas para entidades.
- `Models.EntityReference`: Define relaciones entre entidades (claves foráneas).

2. Gestión de usuarios y seguridad:

- `Models.User`: Representa usuarios del sistema con propiedades como nombre de usuario, contraseña, email y estado de administrador.
- `Models.Role`: Define roles de usuario con permisos.
- `Models.UserRole`: Vincula usuarios con roles (relación muchos a muchos).
- `Models.EntityPermission`: Define permisos para entidades.
- `Models.EntityAttributePermission`: Define permisos para atributos específicos de entidades.
- `Models.RowLevelPermission`: Implementa seguridad a nivel de fila.

3. Personalización y localización:

- `Models.CustomType`: Define tipos de datos personalizados.
- `Models.EnumType`: Define tipos enumerados con valores, colores e iconos.
- `Models.Personalization`: Maneja la personalización del sistema incluyendo colores, logos y configuraciones de idioma.
- Varias entidades de Etiquetas (ej. `EntityLabel`, `CustomTypeLabel`) para soporte multilingüe.

4. Lógica de negocio:

- `Models.Function`: Representa funciones de base de datos y procedimientos almacenados.
- `Models.EntityTrigger`: Define triggers en entidades.
- `Models.EntityCustomTrigger`: Triggers personalizados para entidades.

5. UI y navegación:

- `Models.Dashboard`: Define dashboards del sistema.
- `Models.EntityTab`: Organiza atributos de entidades en pestañas.
- `Models.EntityGroup`: Agrupa atributos de entidades.
- `Models.WebPackage`: Gestiona paquetes web.
- `Models.WebRedirection`: Maneja redirecciones de URL.

6. Inteligencia y análisis:

- `Models.IntelligenceModel`: Define modelos de machine learning.
- `Models.IntelligenceFeature`: Características utilizadas en modelos de inteligencia.
- `Models.IntelligenceMetric`: Métricas para modelos de inteligencia.

7. Configuración del sistema:

- `Models.Variable`: Variables del sistema.
- `Models.Application`: Configuración de aplicaciones.
- `Models.PythonPackage`: Dependencias de paquetes Python.
- `Models.ErrorMessage`: Mensajes de error personalizados.

Relaciones principales:

1. Las entidades tienen atributos, claves y referencias
2. Los usuarios tienen roles y permisos
3. Las entidades pueden tener triggers y triggers personalizados
4. La mayoría de las entidades tienen entidades de etiquetas asociadas para localización
5. Las entidades pueden organizarse en pestañas y grupos
6. Los modelos de inteligencia están vinculados a entidades y sus atributos
