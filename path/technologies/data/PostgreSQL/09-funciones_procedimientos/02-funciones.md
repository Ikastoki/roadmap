# Funciones en PL/pgSQL con `CREATE FUNCTION`

Una funcion es un bloque de codigo almacenado en la base de datos que acepta parametros, ejecuta logica y devuelve un valor. Las funciones se invocan como parte de una expresion SQL (`SELECT funcion(...)`) y, por ser deterministicas por defecto en ciertos contextos, pueden optimizarse mediante caching de planes de ejecucion o utilizarse en indices funcionales.

## Sintaxis basica de `CREATE FUNCTION`

```sql
CREATE OR REPLACE FUNCTION calcular_descuento(
    precio_base numeric,
    porcentaje numeric DEFAULT 10
)
RETURNS numeric
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN precio_base * (1 - porcentaje / 100);
END;
$$;
```

La clausula `LANGUAGE` especifica el lenguaje procedural (`plpgsql`, `sql`, `plpython`, etc.). `AS $$ ... $$` delimita el cuerpo. `CREATE OR REPLACE` actualiza la funcion si ya existe, preservando los permisos asignados.

## Tipos de retorno

### Escalar

```sql
RETURNS integer
RETURNS text
RETURNS boolean
```

Devuelve un unico valor del tipo especificado. Es el caso mas simple y comun.

### `RETURNS TABLE`

```sql
CREATE FUNCTION usuarios_activos()
RETURNS TABLE(id integer, nombre text, email text)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
    SELECT id, nombre, email FROM usuarios WHERE activo = true;
END;
$$;
```

Define una funcion tabla: devuelve un conjunto de filas con una estructura fija. Se usa con `RETURN QUERY` o `RETURN NEXT`.

### `SETOF` tipo

```sql
CREATE FUNCTION listar_nombres()
RETURNS SETOF text
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY SELECT nombre FROM usuarios;
END;
$$;
```

`SETOF` devuelve un conjunto de valores de un mismo tipo (equivalente a `TABLE` con una sola columna). Es util para funciones que generan listas planas.

## Parametros: `IN`, `OUT`, `INOUT`, `VARIADIC`

### `IN` (por defecto)

El parametro de entrada clasico. Se pasa por valor y no se puede modificar.

```sql
CREATE FUNCTION duplicar(IN valor integer) RETURNS integer ...
```

### `OUT` y `INOUT`

Los parametros `OUT` permiten devolver valores sin usar `RETURNS`. PostgreSQL construye automaticamente el tipo de retorno basandose en los parametros `OUT`.

```sql
CREATE FUNCTION dividir(
    IN a integer,
    IN b integer,
    OUT cociente integer,
    OUT resto integer
)
LANGUAGE plpgsql
AS $$
BEGIN
    cociente := a / b;
    resto := a % b;
END;
$$;
```

`INOUT` combina entrada y salida: el parametro recibe un valor y puede modificarse dentro de la funcion.

```sql
CREATE FUNCTION incrementar(INOUT valor integer, IN cantidad integer)
LANGUAGE plpgsql
AS $$
BEGIN
    valor := valor + cantidad;
END;
$$;
```

### `VARIADIC`

Acepta un numero variable de argumentos del mismo tipo, similar a los *variadic args* de otros lenguajes.

```sql
CREATE FUNCTION sumar_varios(VARIADIC valores numeric[])
RETURNS numeric
LANGUAGE plpgsql
AS $$
DECLARE
    total numeric := 0;
BEGIN
    FOR i IN 1 .. array_length(valores, 1) LOOP
        total := total + valores[i];
    END LOOP;
    RETURN total;
END;
$$;
```

## Valores por defecto

Los parametros pueden tener valores por defecto, lo que permite omitirlos en la invocacion.

```sql
CREATE FUNCTION configurar(
    host text DEFAULT 'localhost',
    puerto integer DEFAULT 5432
) RETURNS text ...
```

Los valores por defecto se evalúan en el momento de la llamada y pueden ser expresiones.

## Determinismo: `IMMUTABLE`, `STABLE`, `VOLATILE`

Estas clausulas informan al optimizador sobre el comportamiento de la funcion respecto a los datos:

- `IMMUTABLE`: la funcion siempre devuelve el mismo resultado con los mismos argumentos, incluso entre sesiones. Permite usarla en indices y en `WHERE` con optimizaciones agresivas.
- `STABLE`: dentro de una misma transaccion, devuelve el mismo resultado con los mismos argumentos. No permite usarla en indices, pero si en `WHERE`.
- `VOLATILE` (por defecto): la funcion puede devolver resultados distintos en cada llamada, incluso con los mismos argumentos (ej. `now()`, `random()`).

```sql
CREATE FUNCTION suma(a integer, b integer)
RETURNS integer
LANGUAGE plpgsql
IMMUTABLE
AS $$
BEGIN
    RETURN a + b;
END;
$$;
```

> Marcar una funcion como `IMMUTABLE` cuando en realidad accede a tablas que pueden cambiar produce resultados incorrectos. Es responsabilidad del desarrollador asegurar la veracidad de la declaracion.

