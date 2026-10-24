# Tipos numericos y de texto

PostgreSQL ofrece una amplia gama de tipos de datos numericos y textuales, cada uno disenado para un equilibrio especifico entre precision, rango y espacio de almacenamiento. La eleccion correcta del tipo de dato no solo afecta el rendimiento, sino que tambien previene errores semanticos: almacenar un precio como `REAL` en lugar de `NUMERIC` puede introducir errores de redondeo acumulativos inaceptables en aplicaciones financieras.

## Tipos numericos

### Enteros

Los tipos enteros almacenan numeros sin parte fraccionaria. PostgreSQL ofrece tres variantes que se diferencian en el rango y el espacio en disco:

| Tipo | Bytes | Rango |
|------|-------|-------|
| `SMALLINT` | 2 | -32.768 a 32.767 |
| `INTEGER` | 4 | -2.147.483.648 a 2.147.483.647 |
| `BIGINT` | 8 | -9.223.372.036.854.775.808 a 9.223.372.036.854.775.807 |

```sql
CREATE TABLE ejemplo_enteros (
    id          INTEGER PRIMARY KEY,
    poblacion   BIGINT,
    codigo_postal SMALLINT
);
```

### SERIAL y tipos autoincrementales

`SERIAL`, `BIGSERIAL` y `SMALLSERIAL` son atajos sintacticos que crean una columna entera con una secuencia asociada. Sin embargo, desde PostgreSQL 10 se recomienda usar `GENERATED AS IDENTITY`, que sigue el estandar SQL y ofrece mejor control.

```sql
CREATE TABLE ejemplo_serial (
    id SERIAL PRIMARY KEY
);
-- Equivalente a:
CREATE SEQUENCE ejemplo_serial_id_seq;
CREATE TABLE ejemplo_serial (
    id INTEGER DEFAULT nextval('ejemplo_serial_id_seq') PRIMARY KEY
);
ALTER SEQUENCE ejemplo_serial_id_seq OWNED BY ejemplo_serial.id;
```

### DECIMAL y NUMERIC

`DECIMAL` y `NUMERIC` son sinonimos en PostgreSQL. Almacenan numeros con precision exacta, ideales para cantidades monetarias y calculos que no toleran errores de redondeo. Se definen con precision (numero total de digitos) y escala (digitos a la derecha del punto decimal).

```sql
CREATE TABLE productos (
    precio    NUMERIC(10,2) NOT NULL,  -- 10 digitos, 2 decimales (99999999.99)
    impuesto  NUMERIC(5,3)             -- 5 digitos, 3 decimales (99.999)
);
```

Internamente, `NUMERIC` almacena cada cuatro digitos en un bloque de 16 bits, lo que implica mayor costo computacional que los tipos de punto flotante. Para operaciones masivas donde la exactitud no es critica, `DOUBLE PRECISION` es mas rapido.

### REAL y DOUBLE PRECISION

Tipos de punto flotante de precision variable segun el estandar IEEE 754:

| Tipo | Bytes | Precision | Rango aprox. |
|------|-------|-----------|--------------|
| `REAL` | 4 | 6 digitos decimales | 1E-37 a 1E+37 |
| `DOUBLE PRECISION` | 8 | 15 digitos decimales | 1E-307 a 1E+308 |

```sql
CREATE TABLE medidas (
    temperatura REAL,
    frecuencia DOUBLE PRECISION
);
```

> Los tipos de punto flotante pueden producir errores de redondeo. Por ejemplo, `0.1 + 0.2` con `DOUBLE PRECISION` no es exactamente `0.3`. Para aplicaciones financieras, usar siempre `NUMERIC`.

### MONEY

El tipo `MONEY` almacena cantidades monetarias con una precision fija determinada por la configuracion regional (`lc_monetary`). Su principal ventaja es el formateo automatico al mostrar los datos, pero el calculo exacto puede ser dificil de controlar.

```sql
CREATE TABLE facturas (
    total MONEY
);
INSERT INTO facturas VALUES (123.45);
SELECT total FROM facturas;  -- "$123.45" (o el formato de la locale)
```

### Funciones numericas

PostgreSQL incorpora un amplio repertorio de funciones matematicas:

```sql
SELECT
    abs(-5),                          -- 5
    ceil(4.2), floor(4.2),           -- 5, 4
    round(4.567, 2), trunc(4.567, 2), -- 4.57, 4.56
    power(2, 10), sqrt(25),           -- 1024, 5
    mod(10, 3),                       -- 1
    greatest(1, 5, 3), least(1, 5, 3), -- 5, 1
    random(), pi();                   -- aleatorio entre 0 y 1, 3.14159...
```

## Tipos de texto

PostgreSQL ofrece cuatro tipos principales para almacenar cadenas de caracteres, cada uno con implicaciones de rendimiento y almacenamiento distintas.

### CHAR(n)

Tipo de longitud fija. Las cadenas mas cortas que `n` se rellenan con espacios a la derecha. Si se almacena un valor mas largo, se lanza un error. Es util para codigos de longitud conocida y fija, como ISO de paises o codigos postales.

```sql
CREATE TABLE paises (
    iso CHAR(2) PRIMARY KEY,  -- "ES", "US", "DE"
    nombre TEXT NOT NULL
);
```

### VARCHAR(n)

Tipo de longitud variable con limite. Solo almacena los caracteres realmente introducidos, sin relleno. Es la opcion mas comun para cadenas de longitud variable pero acotada, como nombres de persona o correos electronicos.

```sql
CREATE TABLE usuarios (
    email VARCHAR(255) UNIQUE NOT NULL,
    nombre VARCHAR(100)
);
```

### TEXT

Tipo de longitud variable sin limite practico (maximo 1 GB por valor). Es funcionalmente identico a `VARCHAR` sin argumento de longitud; de hecho, en PostgreSQL no hay diferencia de rendimiento entre `TEXT` y `VARCHAR(n)`.

```sql
CREATE TABLE articulos (
    contenido TEXT
);
```

> En PostgreSQL, `VARCHAR(n)` y `TEXT` se almacenan de forma identica. La unica diferencia es que `VARCHAR(n)` impone una restriccion `CHECK` de longitud. Muchos desarrolladores prefieren `TEXT` por simplicidad y anaden la restriccion explicitamente si la necesitan.

### BYTEA

Almacena datos binarios (_byte arrays_) como imagenes, archivos o datos cifrados. El limite maximo es 1 GB. La salida por defecto es hexadecimal desde PostgreSQL 9.0, y puede manipularse con las funciones `encode` y `decode`.

```sql
CREATE TABLE archivos (
    nombre VARCHAR(255),
    contenido BYTEA
);
INSERT INTO archivos VALUES ('imagen.png', decode('89504E470D0A1A0A', 'hex'));
```

### Funciones de texto

Las funciones de manipulacion de cadenas son esenciales para la limpieza y transformacion de datos directamente en la base de datos:

```sql
SELECT
    length('PostgreSQL'),                -- 10
    trim('  hola  '),                     -- 'hola'
    ltrim('---hola', '-'),                -- 'hola'
    substring('PostgreSQL' FROM 1 FOR 4), -- 'Post'
    replace('abc-123', '-', '_'),         -- 'abc_123'
    split_part('a,b,c', ',', 2),          -- 'b'
    concat('Hola', ' ', 'Mundo'),         -- 'Hola Mundo'
    'Hola' || ' ' || 'Mundo',             -- 'Hola Mundo'
    upper('hola'), lower('HOLA'),         -- 'HOLA', 'hola'
    initcap('juan perez'),                -- 'Juan Perez'
    left('abcdef', 3), right('abcdef', 3)  -- 'abc', 'def'
    repeat('AB', 3),                      -- 'ABABAB'
    reverse('abc'),                       -- 'cba'
    md5('secreto'),                        -- hash MD5 hexadecimal
    encode('datos', 'base64'),            -- 'ZGF0b3M='
    decode('ZGF0b3M=', 'base64');         -- 'datos'
```

Las funciones de hash como `md5` son utiles para almacenar contrasenas con un hash simple (aunque en produccion se recomienda `pgcrypto` con algoritmos mas seguros como bcrypt o SCRAM). `encode` y `decode` permiten transformar datos binarios a representaciones textuales como hexadecimal o base64.

