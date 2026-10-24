# Cursores y manejo de errores

Dos mecanismos esenciales distinguen al PL/pgSQL de la simple ejecucion de consultas: los cursores permiten navegar fila a fila sin cargar todo el resultado en memoria, y el manejo de errores proporciona la capacidad de reaccionar ante fallos sin que la transaccion entera se desvanezca.

## Cursores explicitos

Un cursor explicito se declara, abre, recorre (fetch) y cierra manualmente. Es util cuando se necesita control fino sobre la iteracion o cuando el conjunto de resultados es muy grande.

### Declaracion y apertura

```sql
DECLARE
    cur_usuario CURSOR FOR SELECT id, nombre FROM usuarios;
    fila        RECORD;
BEGIN
    OPEN cur_usuario;
    LOOP
        FETCH NEXT FROM cur_usuario INTO fila;
        EXIT WHEN NOT FOUND;
        RAISE NOTICE 'Usuario: %', fila.nombre;
    END LOOP;
    CLOSE cur_usuario;
END;
```

### `FETCH` y `CLOSE`

`FETCH` avanza el cursor y carga la siguiente fila. Las modalidades incluyen `NEXT` (por defecto), `PRIOR`, `FIRST`, `LAST`, `ABSOLUTE n` y `RELATIVE n`.

```sql
FETCH NEXT FROM cur INTO fila;        -- siguiente fila
FETCH PRIOR FROM cur INTO fila;       -- fila anterior
FETCH ABSOLUTE 5 FROM cur INTO fila;  -- fila 5
```

`CLOSE` libera los recursos asociados al cursor. Todo cursor se cierra automaticamente al terminar la transaccion, pero cerrarlo explicitamente es una buena practica.

### Cursores con parametros

```sql
DECLARE
    cur_activos CURSOR (min_edad integer) FOR
        SELECT * FROM usuarios WHERE edad > min_edad;
BEGIN
    OPEN cur_activos(18);
    ...
    CLOSE cur_activos;
END;
```

## Cursores implicitos (`FOR IN SELECT`)

PL/pgSQL abstrae la gestion del cursor cuando se usa `FOR ... IN SELECT ... LOOP`. El motor abre el cursor, itera y lo cierra automaticamente.

```sql
BEGIN
    FOR fila IN SELECT id, nombre FROM usuarios WHERE activo = true LOOP
        RAISE NOTICE 'Procesando: %', fila.nombre;
    END LOOP;
END;
```

Esta variante es mas segura y legible que los cursores explicitos. Debe preferirse siempre que no se necesite control del posicionamiento o paso de parametros dinamicos.

## `REF CURSOR`

`REFCURSOR` permite definir cursores sin ligarlos a una consulta concreta en tiempo de declaracion. Es el equivalente a un puntero a cursor y resulta util para funciones que devuelven cursores como resultado.

```sql
CREATE FUNCTION abrir_consulta(opcion text)
RETURNS REFCURSOR
LANGUAGE plpgsql
AS $$
DECLARE
    ref REFCURSOR;
BEGIN
    IF opcion = 'activos' THEN
        OPEN ref FOR SELECT * FROM usuarios WHERE activo = true;
    ELSE
        OPEN ref FOR SELECT * FROM usuarios WHERE activo = false;
    END IF;
    RETURN ref;
END;
$$;
```

El cursor se consume desde el cliente o desde otro bloque PL/pgSQL mediante `FETCH ALL FROM ref`.

## Manejo de errores: `BEGIN ... EXCEPTION ... END`

PL/pgSQL permite capturar errores dentro de un sub-bloque con la clausula `EXCEPTION`. Cuando ocurre una excepcion en la seccion `BEGIN`, el control salta al bloque `EXCEPTION`, donde se puede analizar el error y decidir como proseguir.

```sql
BEGIN
    INSERT INTO logs VALUES ('iniciando proceso...');
    PERFORM operacion_riesgosa();
EXCEPTION
    WHEN division_by_zero THEN
        RAISE NOTICE 'Division por cero ignorada';
    WHEN unique_violation THEN
        RAISE NOTICE 'Registro duplicado, se continua';
    WHEN OTHERS THEN
        RAISE NOTICE 'Error inesperado: %', SQLERRM;
END;
```

Cuando se captura una excepcion, los cambios realizados dentro del sub-bloque `BEGIN ... EXCEPTION` se revierten (pero no los del bloque exterior). Esto permite aislar operaciones peligrosas sin perder el trabajo previo.

### `SQLSTATE` y `SQLERRM`

Dentro del bloque `EXCEPTION`, dos variables predefinidas contienen informacion del error:

- `SQLSTATE`: codigo de error de cinco caracteres (ej. `'23505'` para `unique_violation`).
- `SQLERRM`: mensaje de error legible.

```sql
EXCEPTION
    WHEN OTHERS THEN
        RAISE NOTICE 'Codigo: %, Mensaje: %', SQLSTATE, SQLERRM;
```

> No todos los errores se pueden capturar. Los errores graves como `OUT_OF_MEMORY` o `FATAL` terminan la transaccion inmediatamente sin pasar por `EXCEPTION`.

## `RAISE`: notificaciones y errores

`RAISE` emite mensajes en distintos niveles de severidad. Es la herramienta principal para depuracion y para informar al cliente.

```sql
RAISE NOTICE 'Valor actual: %', variable;
RAISE WARNING 'Configuracion no encontrada, usando defecto';
RAISE EXCEPTION 'No se puede procesar el pago: %', motivo;
```

Los niveles disponibles son, de menor a mayor severidad:

- `DEBUG`: solo visible si el nivel de `client_min_messages` lo permite.
- `LOG`: se escribe en el log del servidor.
- `NOTICE`: se muestra al cliente por defecto. Tipico para mensajes informativos.
- `WARNING`: advierte de condiciones anormales no criticas.
- `EXCEPTION`: lanza un error y aborta la transaccion (a menos que se capture).

### `RAISE EXCEPTION` con codigo personalizado

```sql
RAISE EXCEPTION 'Saldo insuficiente'
    USING HINT = 'Revise el saldo disponible',
          ERRCODE = 'P0001';
```

La clausula `USING` permite anadir datos estructurados (`HINT`, `DETAIL`, `ERRCODE`, `SCHEMA`, `TABLE`, `COLUMN`) que el cliente puede interpretar.

> `RAISE EXCEPTION` sin captura deshace la transaccion actual. Si se necesita notificar un error sin perder la transaccion, debe lanzarse dentro de un bloque `BEGIN ... EXCEPTION` que lo capture y lo maneje.

