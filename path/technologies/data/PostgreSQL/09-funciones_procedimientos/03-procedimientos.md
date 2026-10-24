# Procedimientos almacenados con `CREATE PROCEDURE`

Los procedimientos aparecieron en PostgreSQL 11 para cubrir un vacio que las funciones no podian llenar: la gestion explicita de transacciones. Mientras que una funcion se ejecuta dentro de la transaccion de la consulta que la invoca, un procedimiento puede iniciar, confirmar o deshacer transacciones por su cuenta. Esto es fundamental para operaciones ETL, procesamiento por lotes o cualquier tarea que requiera commits intermedios.

## Sintaxis de `CREATE PROCEDURE`

```sql
CREATE OR REPLACE PROCEDURE migrar_datos()
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO historial SELECT * FROM datos_temporales;
    COMMIT;
END;
$$;
```

A diferencia de `CREATE FUNCTION`, `CREATE PROCEDURE` no usa `RETURNS` (un procedimiento no devuelve un valor en el sentido relacional). La invocacion se hace con `CALL` en lugar de `SELECT`.

```sql
CALL migrar_datos();
```

## Diferencias clave entre function y procedure

| Aspecto               | Function                          | Procedure                      |
|-----------------------|-----------------------------------|--------------------------------|
| Retorno               | Siempre devuelve algo (`RETURNS`) | No tiene `RETURNS`             |
| Invocacion            | `SELECT funcion()`                | `CALL procedimiento()`         |
| Transacciones         | No puede hacer `COMMIT`/`ROLLBACK`| Puede gestionar transacciones  |
| Uso en SQL            | Dentro de `SELECT`, `WHERE`, etc. | Solo con `CALL`                |
| Parametros `OUT`      | Si                                | Si                             |
| Llamada desde funcion | Si                                 | No                             |

`CALL` es una sentencia independiente; no se puede usar un procedimiento dentro de una consulta `SELECT`. Si se necesita un valor de retorno, debe usarse un parametro `INOUT`.

## Transacciones en procedimientos

Un procedimiento puede ejecutar `COMMIT` y `ROLLBACK`, algo prohibido dentro de funciones. Esto permite procesar lotes grandes sin mantener una transaccion abierta durante todo el proceso.

```sql
CREATE PROCEDURE procesar_lote(limite integer)
LANGUAGE plpgsql
AS $$
DECLARE
    fila RECORD;
BEGIN
    FOR fila IN SELECT * FROM cola_pendiente LIMIT limite LOOP
        INSERT INTO procesados VALUES (fila.id, fila.dato);
        DELETE FROM cola_pendiente WHERE id = fila.id;
        COMMIT;
    END LOOP;
END;
$$;
```

Cada `COMMIT` libera locks y persiste los cambios. Si ocurre un error en una iteracion posterior, las anteriores ya estan consolidadas.

> Un procedimiento que ejecuta `COMMIT` no puede ejecutarse dentro de una transaccion abierta por el llamante. PostgreSQL lo detecta y lanza un error. Para invocarlo, el bloque exterior debe estar en modo autocommit o sin transaccion explicita.

### `COMMIT` y `ROLLBACK` condicional

```sql
CREATE PROCEDURE transferir(origen integer, destino integer, monto numeric)
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE cuentas SET saldo = saldo - monto WHERE id = origen;
    UPDATE cuentas SET saldo = saldo + monto WHERE id = destino;

    IF (SELECT saldo FROM cuentas WHERE id = origen) < 0 THEN
        ROLLBACK;
        RAISE EXCEPTION 'Saldo insuficiente en cuenta %', origen;
    ELSE
        COMMIT;
    END IF;
END;
$$;
```

Este patron permite validaciones posteriores a las operaciones y decidir si se consolida o se deshace el trabajo.

## Parametros `INOUT` como retorno

Para obtener un valor de salida de un procedimiento se usan parametros `INOUT`:

```sql
CREATE PROCEDURE obtener_estado(
    IN servidor_id integer,
    INOUT estado text DEFAULT 'desconocido'
)
LANGUAGE plpgsql
AS $$
BEGIN
    SELECT status INTO estado FROM servidores WHERE id = servidor_id;
END;
$$;
```

```sql
CALL obtener_estado(1, NULL);
```

> Aunque los procedimientos no tienen `RETURNS`, la presencia de parametros `INOUT` permite emular un valor de retorno. El parametro se pasa como argumento en `CALL` y se modifica dentro del procedimiento.

