# Triggers en PostgreSQL

Las bases de datos transaccionales no solo almacenan datos: reaccionan a ellos. Un trigger es un objeto que asocia un evento de modificacion (INSERT, UPDATE, DELETE) sobre una tabla o vista con una funcion que se ejecuta automaticamente cuando ese evento ocurre. A diferencia de las restricciones de integridad (`CHECK`, `FOREIGN KEY`), que simplemente aceptan o rechazan una operacion, los triggers permiten ejecutar logica arbitraria en respuesta a cambios: registrar auditoria, validar reglas de negocio complejas, sincronizar tablas derivadas o modificar los datos antes de que se persistan.

PostgreSQL es particularmente potente en este ambito porque soporta triggers a nivel de fila y de sentencia, en tres momentos distintos (BEFORE, AFTER, INSTEAD OF), y con la capacidad de inspeccionar tanto el estado anterior como el posterior de la fila afectada.

## Evento, condicion y accion

Conceptualmente un trigger se compone de tres partes:

- **Evento**: la operacion DML que dispara el trigger (`INSERT`, `UPDATE`, `DELETE` o `TRUNCATE`). Pueden combinarse varios eventos en un mismo trigger.
- **Condicion** (opcional): una expresion booleana evaluada antes de ejecutar la accion. Si la condicion se evalua a `false`, la accion se omite. En PostgreSQL se implementa mediante la clausula `WHEN`.
- **Accion**: la funcion (o bloque de sentencias) que se ejecuta. Debe ser una funcion definida que retorne el tipo `TRIGGER`.

```sql
CREATE TRIGGER nombre_trigger
    {BEFORE | AFTER | INSTEAD OF} {evento [OR evento...]}
    ON tabla
    [FOR EACH {ROW | STATEMENT}]
    [WHEN (condicion)]
    EXECUTE FUNCTION nombre_funcion_trigger();
```

La funcion asociada debe estar escrita en PL/pgSQL (o cualquier otro lenguaje procedural soportado por PostgreSQL) y debe retornar obligatoriamente el tipo `TRIGGER`.

## BEFORE, AFTER e INSTEAD OF

El momento de ejecucion determina que informacion esta disponible y que efectos puede tener el trigger.

### BEFORE

El trigger se ejecuta **antes** de que la operacion DML toque la tabla. Esto permite modificar los datos que se van a insertar o actualizar, o incluso cancelar la operacion lanzando una excepcion o retornando `NULL`.

```sql
CREATE TRIGGER trg_before_insert
    BEFORE INSERT ON usuarios
    FOR EACH ROW
    EXECUTE FUNCTION validar_email();
```

Es el unico tipo de trigger que puede modificar `NEW` antes de que los datos se escriban en disco. Tambien es el adecuado para validaciones que deben impedir la operacion.

### AFTER

El trigger se ejecuta **despues** de que la operacion se haya completado y los datos esten en la tabla. No puede modificar `NEW` (la modificacion ya ocurrio), pero tiene acceso completo a los datos ya persistidos.

```sql
CREATE TRIGGER trg_after_insert
    AFTER INSERT ON usuarios
    FOR EACH ROW
    EXECUTE FUNCTION registrar_auditoria();
```

Es el momento natural para registrar historicos, enviar notificaciones o sincronizar tablas secundarias. La operacion ya se ha realizado con exito, por lo que el trigger no puede revertirla (aunque puede lanzar una excepcion que haga rollback de la transaccion completa).

### INSTEAD OF

Exclusivo de **vistas**. Reemplaza por completo la operacion DML sobre la vista. Las vistas no son directamente modificables en PostgreSQL a menos que tengan reglas o triggers INSTEAD OF.

```sql
CREATE TRIGGER trg_insert_vista
    INSTEAD OF INSERT ON vista_usuarios_activos
    FOR EACH ROW
    EXECUTE FUNCTION insertar_en_usuarios();
```

Es la herramienta fundamental para hacer que las vistas sean actualizables cuando la logica de negocio requiere distribuir los datos entre varias tablas subyacentes.

## FOR EACH ROW vs FOR EACH STATEMENT

El alcance del trigger define cuantas veces se ejecuta la funcion asociada.

### FOR EACH ROW

La funcion se ejecuta **una vez por cada fila** afectada por la sentencia. Es el modo mas comun y el unico que tiene acceso a `NEW` y `OLD`.

```sql
CREATE TRIGGER trg_each_row
    AFTER UPDATE ON pedidos
    FOR EACH ROW
    EXECUTE FUNCTION notificar_cambio();
```

Si una sentencia `UPDATE` modifica 1000 filas, la funcion se ejecuta 1000 veces. Esto puede tener implicaciones de rendimiento.

### FOR EACH STATEMENT

La funcion se ejecuta **una vez por sentencia**, independientemente de cuantas filas haya modificado.

```sql
CREATE TRIGGER trg_each_statement
    AFTER UPDATE ON pedidos
    FOR EACH STATEMENT
    EXECUTE FUNCTION actualizar_resumen();
```

Es util para operaciones de agregacion, actualizacion de tablas resumen o limpieza global tras una operacion masiva. No tiene acceso a `NEW` u `OLD` porque no opera a nivel de fila individual.

> Si no se especifica `FOR EACH`, PostgreSQL asume `FOR EACH STATEMENT` por defecto, lo contrario que otros motores como MySQL.

## NEW y OLD: los registros transicionales

Dentro de la funcion trigger, PostgreSQL expone dos variables especiales del tipo `RECORD` que representan el estado de la fila afectada:

- **`NEW`**: contiene los valores de la fila **nueva** (despues del cambio). Disponible en `INSERT` y `UPDATE`. En `BEFORE INSERT` puede modificarse; en `AFTER INSERT` es solo lectura.
- **`OLD`**: contiene los valores de la fila **antigua** (antes del cambio). Disponible en `UPDATE` y `DELETE`. Siempre es solo lectura.

```sql
CREATE FUNCTION registrar_cambio()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO log_cambios(tabla, operacion, id_fila, valor_anterior, valor_nuevo, usuario, momento)
    VALUES (TG_TABLE_NAME, TG_OP, OLD.id, row_to_json(OLD), row_to_json(NEW), current_user, now());
    RETURN NEW;
END;
$$;
```

En `INSERT`, `NEW` contiene la fila que se va a insertar y `OLD` es `NULL`. En `DELETE`, `OLD` contiene la fila eliminada y `NEW` es `NULL`. En `UPDATE`, ambos estan disponibles.

## Variables de contexto en funciones trigger

PostgreSQL inyecta automaticamente varias variables en el entorno de la funcion trigger ademas de `NEW` y `OLD`:

### `TG_OP`

Cadena que indica que operacion disparo el trigger: `'INSERT'`, `'UPDATE'`, `'DELETE'` o `'TRUNCATE'`. Permite que una misma funcion sirva para multiples eventos.

### `TG_TABLE_NAME`

Nombre de la tabla sobre la que se disparo el trigger. Util para funciones genericas que registran auditoria en multiples tablas.

### `TG_TABLE_SCHEMA`

Esquema al que pertenece la tabla. Combinada con `TG_TABLE_NAME` permite referenciar la tabla de forma completamente cualificada.

### `TG_WHEN`

Cadena que indica el momento de ejecucion: `'BEFORE'`, `'AFTER'` o `'INSTEAD OF'`.

### `TG_LEVEL`

`'ROW'` o `'STATEMENT'` segun el alcance del trigger.

### `TG_RELID`

OID de la tabla sobre la que se disparo el trigger.

### `TG_NAME`

Nombre del trigger que se esta ejecutando.

```sql
CREATE FUNCTION log_generico()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    RAISE NOTICE 'Trigger % disparado en %.% por operacion % en momento % nivel %',
        TG_NAME, TG_TABLE_SCHEMA, TG_TABLE_NAME, TG_OP, TG_WHEN, TG_LEVEL;
    RETURN NEW;
END;
$$;
```

## Retorno de la funcion trigger: NEW, OLD, NULL

El valor que la funcion devuelve determina el comportamiento posterior del trigger y de la operacion DML.

### `RETURN NEW`

- En `BEFORE INSERT` y `BEFORE UPDATE`: indica a PostgreSQL que continue con la operacion usando los valores de `NEW` (posiblemente modificados por la funcion).
- En `AFTER INSERT` y `AFTER UPDATE`: el valor de retorno se ignora practicamente, pero debe devolverse `NEW` por convencion.
- En `DELETE`: debe retornarse `OLD`.

### `RETURN OLD`

- En `BEFORE DELETE`: permite que la operacion continue (el borrado se ejecuta). Devolver `NULL` en `BEFORE DELETE` cancelaria el borrado.
- En `AFTER DELETE`: se devuelve `OLD` por convencion.

### `RETURN NULL`

- En `BEFORE INSERT`, `BEFORE UPDATE` o `BEFORE DELETE`: **cancela la operacion** para la fila actual. La fila no se inserta, modifica ni elimina. No se produce error, simplemente la fila se salta.
- En `AFTER` triggers: el valor de retorno no afecta a la operacion, pero debe devolverse `NEW` u `OLD` segun el caso. Devolver `NULL` en un `AFTER` trigger no deshace la operacion, pero es una mala practica.

> Devolver `NULL` en un `BEFORE INSERT` es la forma silenciosa de impedir que una fila se inserte sin lanzar una excepcion. Si se necesita rechazar la operacion con un mensaje de error, debe usarse `RAISE EXCEPTION`.

```sql
CREATE FUNCTION evitar_nulos()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF NEW.nombre IS NULL THEN
        RETURN NULL;  -- la fila no se inserta, sin error
    END IF;
    RETURN NEW;
END;
$$;
```

