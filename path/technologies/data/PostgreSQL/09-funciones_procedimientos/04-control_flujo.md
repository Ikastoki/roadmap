# Control de flujo en PL/pgSQL

La potencia de PL/pgSQL reside en sus estructuras de control de flujo: condicionales, bucles y saltos logicos que permiten escribir logica tan compleja como la de cualquier lenguaje de proposito general. Sin ellas, el codigo procedural se limitaria a secuencias lineales de sentencias SQL.

## Condicionales: `IF` y `CASE`

### `IF ... THEN ... ELSIF ... ELSE ... END IF`

```sql
IF saldo < 0 THEN
    RAISE NOTICE 'Cuenta en negativo';
ELSIF saldo = 0 THEN
    RAISE NOTICE 'Cuenta a cero';
ELSE
    RAISE NOTICE 'Cuenta con saldo positivo';
END IF;
```

`ELSIF` (una sola palabra, no `ELSEIF` ni `ELSE IF`) permite encadenar condiciones. Se evaluan en orden y se ejecuta la primera que sea verdadera. Si ninguna lo es y existe `ELSE`, se ejecuta ese bloque.

### `CASE ... WHEN ... THEN ... END CASE`

```sql
CASE estado
    WHEN 'activo' THEN
        PERFORM actualizar_online(id);
    WHEN 'inactivo' THEN
        PERFORM marcar_offline(id);
    ELSE
        RAISE NOTICE 'Estado desconocido: %', estado;
END CASE;
```

Existe tambien la forma *buscada* sin expresion inicial:

```sql
CASE
    WHEN temperatura > 30 THEN
        RAISE NOTICE 'Alerta calor';
    WHEN temperatura < 0 THEN
        RAISE NOTICE 'Alerta hielo';
    ELSE
        RAISE NOTICE 'Temperatura normal';
END CASE;
```

## Bucles

### `LOOP ... END LOOP`

El bucle infinito (debe incluir una salida explicita).

```sql
LOOP
    PERFORM tarea_programada();
    EXIT WHEN pg_sleep(1);
END LOOP;
```

### `WHILE ... LOOP ... END LOOP`

```sql
WHILE contador > 0 LOOP
    RAISE NOTICE 'Iteracion %', contador;
    contador := contador - 1;
END LOOP;
```

### `FOR ... IN ... LOOP ... END LOOP`

Bucle con rango numerico (equivalente a `for i in range(...)` de Python).

```sql
FOR i IN 1..10 LOOP
    RAISE NOTICE 'Numero: %', i;
END LOOP;
```

Variante descendente con `REVERSE`:

```sql
FOR i IN REVERSE 10..1 LOOP
    RAISE NOTICE 'Contando atras: %', i;
END LOOP;
```

El paso es siempre 1, no configurable.

### `FOREACH ... IN ARRAY ... LOOP ... END LOOP`

Itera sobre los elementos de un array multidimensional.

```sql
DECLARE
    valores integer[] := ARRAY[10, 20, 30, 40];
    v integer;
BEGIN
    FOREACH v IN ARRAY valores LOOP
        RAISE NOTICE 'Elemento: %', v;
    END LOOP;
END;
```

Para arrays multidimensionales, `SLICE N` permite iterar sub-arrays completos.

```sql
FOREACH v SLICE 1 IN ARRAY matriz LOOP
    RAISE NOTICE 'Fila: %', v;
END LOOP;
```

## Saltos y salidas

### `EXIT WHEN`

Sale del bucle inmediatamente cuando se cumple una condicion.

```sql
FOR i IN 1..100 LOOP
    EXIT WHEN i > 50;
    RAISE NOTICE '%', i;
END LOOP;
```

### `CONTINUE WHEN`

Salta a la siguiente iteracion sin ejecutar el resto del cuerpo.

```sql
FOR i IN 1..10 LOOP
    CONTINUE WHEN i % 2 = 0;
    RAISE NOTICE 'Impar: %', i;
END LOOP;
```

## Return en funciones

### `RETURN`

Devuelve un valor escalar y termina la funcion.

```sql
RETURN precio_total * 1.21;
```

### `RETURN NEXT`

Acumula una fila en el resultado de una funcion que devuelve `SETOF` o `TABLE`. No termina la funcion; el resultado se construye incrementalmente.

```sql
CREATE FUNCTION pares(hasta integer)
RETURNS SETOF integer
LANGUAGE plpgsql
AS $$
BEGIN
    FOR i IN 1..hasta LOOP
        IF i % 2 = 0 THEN
            RETURN NEXT i;
        END IF;
    END LOOP;
END;
$$;
```

### `RETURN QUERY`

Ejecuta una consulta y anade todas sus filas al resultado.

```sql
RETURN QUERY SELECT id, nombre FROM usuarios WHERE activo = true;
```

Puede combinarse con `RETURN QUERY EXECUTE` para consultas dinamicas.

```sql
RETURN QUERY EXECUTE format('SELECT * FROM %I WHERE activo = true', nombre_tabla);
```

> `RETURN NEXT` y `RETURN QUERY` no detienen la funcion. Si se necesita salir tras devolver, debe usarse un `RETURN` desnudo o `RETURN NULL`.

