### Resumen: campos multi‑referencia y restricciones FK con arrays

#### Contexto del error

Al crear una referencia desde el diagrama E/R (EntityList) para un campo multi‑referencia, aparece este error de PostgreSQL:

```text
ERROR: foreign key constraint "id_users" cannot be implemented
Detail: Key columns "id_users" and "id" are of incompatible types: integer[] and integer.
Where: SQL statement "ALTER TABLE IF EXISTS "test"."references" ADD CONSTRAINT "id_users" FOREIGN KEY ("id_users") REFERENCES "test"."users"("id") ON DELETE CASCADE DEFERRABLE INITIALLY IMMEDIATE"
PL/pgSQL function "Models".synchronizer() line 886 at EXECUTE
```

#### Causa raíz

- PostgreSQL no permite crear una clave foránea sobre una columna de tipo array (por ejemplo `integer[]`) que apunte a una clave primaria escalar (`integer`).
- El sincronizador intenta crear una FK clásica incluso cuando el atributo referenciador es `isArray = true`.

#### Dónde sucede en el código

- Generación de FKs en el sincronizador: `com.niledb.platform/misc/model/sql/deltas/138.sql` (bloque que construye el `ALTER TABLE ... ADD CONSTRAINT ... FOREIGN KEY ...`).
- Modelo local con indicador de array: `airflows-admin-tool/src/data/mocks/localModelJson.json` (propiedad `isArray`).

#### Opciones de solución (con pros y contras)

- 1) Omitir la creación de FKs cuando el atributo referenciador sea array
  - Pros: desbloquea el sincronizado de inmediato; cambio mínimo.
  - Contras: sin integridad referencial a nivel BD; las opciones de borrado en cascada no se aplican por el motor.

- 2) Mantener arrays y reforzar integridad con triggers
  - Idea: validar en `INSERT/UPDATE` que todos los IDs de `unnest(array_col)` existan; en `DELETE` de la tabla referenciada, podar IDs eliminados en arrays; opcionalmente, reemplazar IDs si cambian.
  - Pros: preserva UX con arrays; emula cascadas e integridad.
  - Contras: más complejidad y coste de mantenimiento; no es un FK real (herramientas de BD no lo verán); hay que cuidar rendimiento y diferibilidad.

- 3) Normalizar con tabla de enlace (muchos‑a‑muchos)
  - Idea: reemplazar `integer[]` por una tabla `link` (por ejemplo `entity__users(entity_id, user_id)`) con FKs reales y cascadas.
  - Pros: integridad y escalabilidad estándar; soporte óptimo de índices y consultas.
  - Contras: cambios en DSL/UI y migraciones; adaptar consultas y el E/R.

- 4) Híbrido: tabla de enlace como fuente de verdad + vista/array para la UI
  - Idea: la verdad en la tabla `link` y exponer un array agregado (vista o columna sincronizada). Lectura como array; escritura vía triggers o hacer el array de sólo lectura.
  - Pros: integridad real con mínima fricción en UI si el array es read‑only.
  - Contras: implementación más compleja; riesgo de ciclos si se sincroniza en ambos sentidos.

- 5) Aceptar no tener FK y validar sólo en aplicación
  - Pros: esfuerzo mínimo.
  - Contras: riesgo de desalineación de datos; sin cascadas; deuda técnica.

#### Recomendación

- Corto plazo: añadir una salvaguarda en el sincronizador para NO generar FKs cuando alguno de los atributos de la referencia tenga `isArray = true`, y emitir un `NOTICE` explicativo durante el sync.
- Próximo paso si se requiere integridad/cascadas con arrays: implementar triggers de validación y poda/actualización de IDs.
- Largo plazo (si la integridad, rendimiento y mantenibilidad son prioritarios): migrar multi‑referencias a una tabla de enlace (opción 3) o al enfoque híbrido (opción 4).


