# Ejemplos practicos y gestion de triggers

Los triggers resuelven problemas recurrentes en el desarrollo de bases de datos: mantener pistas de auditoria, garantizar integridad semantica que las constraints no cubren, sincronizar tablas derivadas y automatizar campos de metadatos temporales. En esta seccion se recorren los casos de uso mas habituales junto con las operaciones de administracion que permiten controlar su ciclo de vida.

## Auditoria: registro de cambios

El caso mas clasico es registrar cada modificacion en una tabla de historico. Una funcion generica que usa las variables de contexto de PostgreSQL puede servir para cualquier tabla.

```sql
CREATE TABLE log_auditoria (
    id          serial PRIMARY KEY,
    tabla       text NOT NULL,
    operacion   text NOT NULL,
    id_fila     integer NOT NULL,
    datos_old   jsonb,
    datos_new   jsonb,
    usuario     text NOT NULL DEFAULT current_user,
    momento     timestamptz NOT NULL DEFAULT now()
);

CREATE FUNCTION audit_trigger()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO log_auditoria(tabla, operacion, id_fila, datos_new)
        VALUES (TG_TABLE_NAME, TG_OP, NEW.id, row_to_json(NEW)::jsonb);
        RETURN NEW;

    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO log_auditoria(tabla, operacion, id_fila, datos_old, datos_new)
        VALUES (TG_TABLE_NAME, TG_OP, OLD.id, row_to_json(OLD)::jsonb, row_to_json(NEW)::jsonb);
        RETURN NEW;

    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO log_auditoria(tabla, operacion, id_fila, datos_old)
        VALUES (TG_TABLE_NAME, TG_OP, OLD.id, row_to_json(OLD)::jsonb);
        RETURN OLD;
    END IF;
END;
$$;
```

Aplicado sobre la tabla `usuarios`:

```sql
CREATE TRIGGER trg_audit_usuarios
    AFTER INSERT OR UPDATE OR DELETE ON usuarios
    FOR EACH ROW
    EXECUTE FUNCTION audit_trigger();
```

> El uso de `jsonb` y `row_to_json` permite almacenar cualquier estructura de fila sin necesidad de definir columnas especificas para cada tabla auditada. La desventaja es que los cambios de esquema no rompen el trigger pero los datos historicos mantienen la estructura antigua.

## Validacion con RAISE EXCEPTION

Cuando una `CHECK` constraint no es suficiente porque la validacion requiere acceso a otras tablas o logica procedural, un trigger BEFORE es la solucion.

```sql
CREATE FUNCTION validar_presupuesto()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
DECLARE
    presupuesto_restante numeric;
BEGIN
    SELECT presupuesto - COALESCE(SUM(gastado), 0)
    INTO presupuesto_restante
    FROM departamentos d
    LEFT JOIN gastos g ON g.departamento_id = d.id
    WHERE d.id = NEW.departamento_id
    GROUP BY d.id;

    IF NEW.importe > presupuesto_restante THEN
        RAISE EXCEPTION 'El departamento % no tiene presupuesto suficiente. Restante: %, solicitado: %',
            NEW.departamento_id, presupuesto_restante, NEW.importe;
    END IF;

    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_check_presupuesto
    BEFORE INSERT OR UPDATE ON gastos
    FOR EACH ROW
    EXECUTE FUNCTION validar_presupuesto();
```

A diferencia de devolver `NULL`, que silenciosamente ignora la fila, `RAISE EXCEPTION` propaga un error que la aplicacion cliente puede capturar y presentar al usuario.

## Sincronizacion de tablas derivadas

Los datos agregados o resumidos que se consultan con frecuencia pueden mantenerse actualizados automaticamente mediante triggers, evitando costosas recomputaciones en tiempo real.

```sql
CREATE TABLE resumen_categorias (
    categoria_id  integer PRIMARY KEY,
    total_productos integer NOT NULL DEFAULT 0,
    stock_total   integer NOT NULL DEFAULT 0,
    ultima_actualizacion timestamptz
);

CREATE FUNCTION sincronizar_resumen()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO resumen_categorias (categoria_id, total_productos, stock_total, ultima_actualizacion)
        VALUES (NEW.categoria_id, 1, NEW.stock, now())
        ON CONFLICT (categoria_id) DO UPDATE SET
            total_productos = resumen_categorias.total_productos + 1,
            stock_total     = resumen_categorias.stock_total + NEW.stock,
            ultima_actualizacion = now();

    ELSIF TG_OP = 'UPDATE' THEN
        UPDATE resumen_categorias SET
            total_productos = total_productos + CASE WHEN NEW.categoria_id <> OLD.categoria_id THEN 1 ELSE 0 END,
            stock_total     = stock_total - OLD.stock + NEW.stock,
            ultima_actualizacion = now()
        WHERE categoria_id = OLD.categoria_id;

        IF NEW.categoria_id <> OLD.categoria_id THEN
            INSERT INTO resumen_categorias (categoria_id, total_productos, stock_total, ultima_actualizacion)
            VALUES (NEW.categoria_id, 1, NEW.stock, now())
            ON CONFLICT (categoria_id) DO UPDATE SET
                total_productos = resumen_categorias.total_productos + 1,
                stock_total     = resumen_categorias.stock_total + NEW.stock,
                ultima_actualizacion = now();
        END IF;

    ELSIF TG_OP = 'DELETE' THEN
        UPDATE resumen_categorias SET
            total_productos = total_productos - 1,
            stock_total     = stock_total - OLD.stock,
            ultima_actualizacion = now()
        WHERE categoria_id = OLD.categoria_id;
    END IF;

    RETURN NULL;  -- trigger AFTER, el valor se ignora
END;
$$;
```

El patron `INSERT ... ON CONFLICT DO UPDATE` (upsert) evita tener que verificar previamente si la fila en el resumen existe.

## Auto-generar created_at y updated_at

PostgreSQL no tiene un equivalente nativo al `ON UPDATE CURRENT_TIMESTAMP` de MySQL. La forma correcta de mantener marcas temporales es mediante triggers.

```sql
CREATE FUNCTION set_timestamps()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.created_at := COALESCE(NEW.created_at, now());
    NEW.updated_at := now();
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_timestamps_usuarios
    BEFORE INSERT OR UPDATE ON usuarios
    FOR EACH ROW
    EXECUTE FUNCTION set_timestamps();
```

- En `INSERT`: `created_at` se asigna solo si no se ha proporcionado explicitamente; `updated_at` se establece siempre.
- En `UPDATE`: `updated_at` se actualiza; `created_at` no se toca.

La tabla correspondiente:

```sql
CREATE TABLE usuarios (
    id         serial PRIMARY KEY,
    nombre     text NOT NULL,
    email      text NOT NULL,
    created_at timestamptz NOT NULL DEFAULT now(),
    updated_at timestamptz NOT NULL DEFAULT now()
);
```

El `DEFAULT now()` en la definicion de la tabla sirve como valor por defecto si el trigger no se ejecuta (por ejemplo, en inserciones directas con `DEFAULT`). No obstante, para total consistencia conviene que el trigger sea la unica fuente de verdad para estos campos.

## Soft delete: borrado logico

En lugar de eliminar registros fisicamente, se marcan como eliminados. Un trigger BEFORE puede interceptar el DELETE y convertirlo en un UPDATE.

```sql
CREATE FUNCTION soft_delete()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE productos SET
        deleted_at = now(),
        deleted_by = current_user
    WHERE id = OLD.id;
    RETURN NULL;  -- cancela el DELETE original
END;
$$;

CREATE TRIGGER trg_soft_delete
    BEFORE DELETE ON productos
    FOR EACH ROW
    EXECUTE FUNCTION soft_delete();
```

Para que el borrado logico sea transparente, suele anadirse una vista que filtre los registros no eliminados y se opera contra ella:

```sql
CREATE VIEW productos_activos AS
SELECT * FROM productos WHERE deleted_at IS NULL;

CREATE RULE _delete AS
    ON DELETE TO productos_activos
    DO INSTEAD
    UPDATE productos SET deleted_at = now() WHERE id = OLD.id;
```

> El soft delete tiene implicaciones en las claves foraneas: si otra tabla referencia `productos`, la restriccion `FOREIGN KEY` no impedira referenciar un producto eliminado logicamente. Esto puede ser deseable o no segun el caso.

## Gestion de triggers

PostgreSQL ofrece comandos especificos para administrar el ciclo de vida de los triggers una vez creados.

### DROP TRIGGER

Elimina un trigger de forma permanente.

```sql
DROP TRIGGER [IF EXISTS] nombre_trigger ON tabla [CASCADE | RESTRICT];
```

```sql
DROP TRIGGER IF EXISTS trg_audit_usuarios ON usuarios;
```

La opcion `CASCADE` propaga el borrado a objetos que dependan del trigger (poco habitual). `RESTRICT` es el comportamiento por defecto e impide el borrado si existen dependencias.

No es posible modificar un trigger directamente. Para cambiarlo debe eliminarse y volverse a crear dentro de una misma transaccion, lo que garantiza que no haya un momento sin el trigger activo:

```sql
BEGIN;
DROP TRIGGER trg_check_presupuesto ON gastos;
CREATE TRIGGER trg_check_presupuesto
    BEFORE INSERT OR UPDATE ON gastos
    FOR EACH ROW
    EXECUTE FUNCTION validar_presupuesto_v2();
COMMIT;
```

### ALTER TRIGGER ... RENAME

El unico aspecto modificable de un trigger existente sin recrearlo es su nombre:

```sql
ALTER TRIGGER nombre_trigger ON tabla RENAME TO nuevo_nombre;
```

### ALTER TABLE ... DISABLE TRIGGER

Desactiva un trigger sin eliminarlo. Sigue existiendo como objeto en el catalogo, pero no se ejecuta.

```sql
ALTER TABLE nombre_tabla DISABLE TRIGGER nombre_trigger;
ALTER TABLE nombre_tabla DISABLE TRIGGER ALL;  -- desactiva todos los triggers de la tabla
```

Es util durante migraciones masivas de datos donde los triggers de auditoria ralentizarian la operacion o inserciones historicas que no deben generar registros de auditoria.

### ALTER TABLE ... ENABLE TRIGGER

Reactiva un trigger previamente desactivado.

```sql
ALTER TABLE nombre_tabla ENABLE TRIGGER nombre_trigger;
ALTER TABLE nombre_tabla ENABLE TRIGGER ALL;
```

```sql
-- Carga masiva sin triggers
ALTER TABLE usuarios DISABLE TRIGGER ALL;
INSERT INTO usuarios SELECT * from usuarios_importados;
ALTER TABLE usuarios ENABLE TRIGGER ALL;
```

### Listar triggers existentes

El catalogo `pg_trigger` contiene la informacion de todos los triggers del sistema. Para obtener una vista legible:

```sql
SELECT
    tgname AS nombre_trigger,
    relname AS tabla,
    pg_get_triggerdef(oid) AS definicion
FROM pg_trigger
JOIN pg_class ON pg_trigger.tgrelid = pg_class.oid
WHERE NOT tgisinternal;
```

O usando el meta-comando de psql:

```sql
\dS tabla
```

La `S` incluye objetos del sistema, lo que garantiza que se muestren los triggers aunque la tabla este en un esquema que no esta en el `search_path`.

