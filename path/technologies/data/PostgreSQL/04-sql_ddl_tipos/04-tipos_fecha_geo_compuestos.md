# Tipos de fecha, red, geo, compuestos y especiales

PostgreSQL es conocido por su riqueza de tipos de datos, que va mucho mas alla de los numeros y las cadenas. El soporte nativo para fechas con zona horaria, direcciones de red, geometria, JSON, arreglos, enumeraciones y rangos permite modelar dominios complejos sin recurrir a capas de abstraccion externas. Esta diversidad tipologica reduce la impedancia entre el modelo relacional y los objetos del dominio de la aplicacion.

## Tipos de fecha y hora

PostgreSQL maneja fechas y horas con una precision de hasta microsegundos (6 digitos fraccionarios) y soporta operaciones aritmeticas directas entre tipos temporales.

### DATE, TIME, TIMESTAMP e INTERVAL

| Tipo | Descripcion | Ejemplo |
|------|-------------|---------|
| `DATE` | Fecha (ano, mes, dia) | `2026-07-28` |
| `TIME` | Hora del dia (sin fecha) | `14:30:00` |
| `TIME WITH TIME ZONE` | Hora con zona horaria | `14:30:00+02` |
| `TIMESTAMP` | Fecha y hora sin zona | `2026-07-28 14:30:00` |
| `TIMESTAMPTZ` | Fecha y hora con zona | `2026-07-28 14:30:00+02` |
| `INTERVAL` | Periodo de tiempo | `2 years 3 months` |

```sql
CREATE TABLE eventos (
    inicio     TIMESTAMP,
    fin        TIMESTAMPTZ,
    duracion   INTERVAL,
    solo_hora  TIME,
    solo_fecha DATE
);
```

> `TIMESTAMPTZ` no almacena la zona horaria en disco. Simplemente convierte la hora proporcionada a UTC internamente y la presenta en la zona horaria configurada por el cliente (`TimeZone`). Esto evita problemas de conversion y husos horarios variables.

### Constantes y literales

```sql
-- Fechas en formato ISO 8601 (recomendado)
DATE '2026-07-28'
TIMESTAMP '2026-07-28 14:30:00'
TIMESTAMPTZ '2026-07-28 14:30:00+02'
INTERVAL '1 year 2 months 3 days 4 hours'
INTERVAL '3 months'  -- simplificado
```

### Funciones de fecha y hora

```sql
SELECT
    now(),                           -- 2026-07-28 14:30:00.123456+02
    current_timestamp,               -- igual que now()
    current_date,                    -- 2026-07-28
    current_time,                    -- 14:30:00.123456+02
    age('2026-07-28', '2000-01-15'), -- 26 years 6 mons 13 days
    extract(YEAR FROM now()),        -- 2026
    date_trunc('month', now()),      -- 2026-07-01 00:00:00+02
    date_part('dow', now()),         -- 2 (martes, 0=domingo)
    to_char(now(), 'YYYY-MM-DD HH24:MI:SS'), -- '2026-07-28 14:30:00'
    to_date('28-07-2026', 'DD-MM-YYYY'),     -- 2026-07-28
    to_timestamp('2026-07-28', 'YYYY-MM-DD'), -- 2026-07-28 00:00:00+02
    make_date(2026, 7, 28),          -- 2026-07-28
    make_time(14, 30, 0);            -- 14:30:00
```

### Operadores aritmeticos

```sql
SELECT
    now() + INTERVAL '1 day',        -- manana a la misma hora
    now() - INTERVAL '3 hours',      -- hace 3 horas
    '2026-07-28'::DATE - '2026-07-01'::DATE,  -- 27 (dias de diferencia)
    '2026-07-01'::DATE + 30;         -- 2026-07-31 (suma de dias)
```

## Tipo booleano

El tipo `BOOLEAN` almacena los valores `TRUE`, `FALSE` y `NULL`. PostgreSQL acepta multiples representaciones en la entrada: `true`, `yes`, `on`, `1` para verdadero, y `false`, `no`, `off`, `0` para falso.

```sql
CREATE TABLE configuracion (
    activo BOOLEAN DEFAULT TRUE,
    visible BOOLEAN NOT NULL
);
INSERT INTO configuracion VALUES (TRUE, 'no');   -- 'no' se interpreta como FALSE
SELECT * FROM configuracion WHERE activo;        -- WHERE activo = TRUE
SELECT * FROM configuracion WHERE NOT activo;    -- WHERE activo = FALSE
```

> `NULL` en un booleano no es ni verdadero ni falso, sino desconocido. Una condicion `WHERE activo` solo devuelve filas donde `activo = TRUE`, excluyendo tanto `FALSE` como `NULL`.

## Tipos de red

PostgreSQL ofrece tipos nativos para direcciones de red y direcciones MAC, con validacion de formato y operadores de inclusion y pertenencia a subred.

### INET y CIDR

`INET` almacena direcciones IPv4 o IPv6 con mascara opcional. `CIDR` es similar pero exige que la mascara sea valida segun el direccionamiento CIDR (los bits de red deben estar a la izquierda).

```sql
CREATE TABLE conexiones (
    ip_origen   INET,
    ip_destino  CIDR
);
INSERT INTO conexiones VALUES
    ('192.168.1.10/24', '10.0.0.0/8'),
    ('::1', '2001:db8::/32');
SELECT * FROM conexiones WHERE ip_origen <<= '192.168.0.0/16';
-- <<= : contenido en la subred
```

### MACADDR y MACADDR8

Almacenan direcciones MAC de 6 y 8 bytes respectivamente.

```sql
CREATE TABLE dispositivos (
    mac MACADDR PRIMARY KEY
);
INSERT INTO dispositivos VALUES ('08:00:2b:01:02:03');
```

## Tipos geometricos

PostgreSQL incluye soporte nativo para tipos geometricos bidimensionales en el plano cartesiano, con operadores de distancia, area, interseccion y contencion.

| Tipo | Descripcion | Representacion |
|------|-------------|----------------|
| `POINT` | Punto (x, y) | `(1.0, 2.0)` |
| `LINE` | Linea infinita | `{1.0, -2.0, 3.0}` |
| `LSEG` | Segmento | `[(1.0, 2.0), (3.0, 4.0)]` |
| `BOX` | Rectangulo | `(3.0, 4.0),(1.0, 2.0)` |
| `PATH` | Polilinea (abierta/cerrada) | `[(1,2),(3,4),(5,6)]` |
| `POLYGON` | Poligono cerrado | `((1,2),(3,4),(5,6))` |
| `CIRCLE` | Circulo | `<(-1,0), 5.0>` |

```sql
CREATE TABLE lugares (
    coordenadas POINT,
    area BOX
);
INSERT INTO lugares VALUES ('(10.5, 20.3)', '((10, 20), (30, 40))');
```

## UUID

El tipo `UUID` almacena identificadores universalmente unicos de 128 bits segun RFC 4122. Se utiliza como clave primaria alternativa a los enteros autoincrementales, especialmente en sistemas distribuidos donde la generacion de identificadores debe ser independiente de una base de datos central.

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE TABLE sesiones (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    token UUID NOT NULL DEFAULT uuid_generate_v4()
);
```

Desde PostgreSQL 13, `gen_random_uuid()` esta disponible sin extensiones y es la funcion recomendada.

## JSON y JSONB

PostgreSQL fue pionero en el soporte nativo de JSON dentro del motor relacional. Ofrece dos tipos:

- `JSON`: almacena el texto JSON literal, conservando espacios, orden de claves y duplicados. Es util solo cuando se necesita preservar la representacion exacta.
- `JSONB`: almacena el JSON en un formato binario descompuesto, eliminando espacios superfluos y duplicados. Permite indexacion GIN y operadores de busqueda eficientes.

```sql
CREATE TABLE registros (
    datos JSONB
);
INSERT INTO registros VALUES ('{"nombre": "Ana", "edad": 30, "tags": ["admin", "user"]}');
```

Operadores basicos:

```sql
SELECT
    datos -> 'nombre',              -- "Ana" (JSONB)
    datos ->> 'nombre',             -- Ana (texto)
    datos #> '{tags, 0}',          -- "admin" (ruta)
    datos @> '{"edad": 30}',       -- true (contiene)
    datos ? 'nombre',              -- true (tiene clave)
    datos || '{"activo": true}';   -- merge
```

## ARRAY

Cualquier tipo de dato en PostgreSQL puede usarse como un arreglo multidimensional. Los arreglos se declaran anadiendo corchetes al tipo base.

```sql
CREATE TABLE inventario (
    etiquetas TEXT[],
    matriz INTEGER[3][3]
);
INSERT INTO inventario VALUES (ARRAY['rojo', 'verde', 'azul'], ARRAY[[1,2,3],[4,5,6],[7,8,9]]);
```

Operaciones comunes:

```sql
SELECT
    array_append(ARRAY[1,2,3], 4),            -- {1,2,3,4}
    array_prepend(0, ARRAY[1,2,3]),           -- {0,1,2,3}
    array_cat(ARRAY[1,2], ARRAY[3,4]),        -- {1,2,3,4}
    array_length(ARRAY[1,2,3], 1),            -- 3
    unnest(ARRAY['a','b','c']),               -- filas: a, b, c
    ARRAY[1,2,3] @> ARRAY[1,3],              -- true (contiene)
    2 = ANY(ARRAY[1,2,3]),                    -- true
    ARRAY[1,2,3] && ARRAY[3,4,5];            -- true (interseccion)
```

## Tipos compuestos: ENUM, COMPOSITE, RANGE, DOMAIN

### ENUM

Los tipos enumerados definen un conjunto fijo de valores. Se almacenan internamente como numeros de 4 bytes para eficiencia.

```sql
CREATE TYPE estado_pedido AS ENUM ('pendiente', 'procesando', 'enviado', 'entregado', 'cancelado');
CREATE TABLE pedidos (
    estado estado_pedido DEFAULT 'pendiente'
);
```

### COMPOSITE (tipo compuesto)

Un tipo compuesto agrupa multiples campos en una sola columna, similar a un registro o una estructura.

```sql
CREATE TYPE direccion AS (
    calle    TEXT,
    ciudad   TEXT,
    codigo_postal CHAR(5)
);
CREATE TABLE clientes (
    id INTEGER PRIMARY KEY,
    domicilio direccion
);
INSERT INTO clientes VALUES (1, ROW('Av. Siempre Viva', 'Springfield', '12345'));
SELECT (domicilio).ciudad FROM clientes;
```

### RANGE

Los tipos rango representan intervalos de valores con limites inferior y superior, inclusivos o exclusivos. PostgreSQL ofrece rangos predefinidos para enteros (`int4range`), enteros largos (`int8range`), numericos (`numrange`), fechas (`daterange`) y timestamps (`tsrange`, `tstzrange`).

```sql
CREATE TABLE reservas (
    sala       INTEGER,
    periodo    tsrange,
    EXCLUDE USING gist (sala WITH =, periodo WITH &&)
);
INSERT INTO reservas VALUES (1, '[2026-07-28 10:00, 2026-07-28 12:00)');
SELECT * FROM reservas WHERE periodo @> '2026-07-28 11:00'::TIMESTAMP;
```

Los operadores de rango incluyen `@>` (contiene), `<@` (contenido en), `&&` (solapamiento), `<<` (estrictamente antes), `>>` (estrictamente despues) y `-|-` (adyacente).

### DOMAIN

Un dominio es un tipo de dato definido por el usuario que anade restricciones a un tipo base. Es una forma de reutilizar reglas de validacion en multiples columnas.

```sql
CREATE DOMAIN email_valido AS TEXT
    CHECK (VALUE ~* '^[A-Za-z0-9._%-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$');

CREATE DOMAIN positivo AS NUMERIC(10,2)
    CHECK (VALUE > 0);

CREATE TABLE usuarios (
    email email_valido NOT NULL,
    saldo positivo DEFAULT 0
);
```

Los dominios se integran con el sistema de tipos y pueden ser usados en cualquier lugar donde se espere un tipo base. Si la restriccion cambia, basta con alterar el dominio y todas las columnas que lo usan se actualizan automaticamente.

