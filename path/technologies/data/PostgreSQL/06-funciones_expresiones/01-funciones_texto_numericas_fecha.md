# Funciones de texto, numericas y de fecha

PostgreSQL ofrece un repertorio extenso de funciones integradas que cubren practicamente cualquier necesidad de transformacion y calculo sobre datos. Estas funciones se ejecutan dentro del motor de base de datos, lo que evita trasladar datos a la capa de aplicacion para operaciones rutinarias y permite construir consultas mas expresivas y eficientes. Comprenderlas es esencial para escribir SQL productivo y evitar logicas innecesarias en codigo externo.

## Funciones de texto

Las funciones de texto permiten manipular cadenas de caracteres: medir longitudes, cambiar mayusculas, extraer porciones, buscar patrones y transformar contenido. PostgreSQL soporta texto con codificacion UTF-8 de forma nativa, por lo que estas funciones respetan caracteres multibyte.

### Longitud y capitalizacion

`length()` devuelve el numero de caracteres de una cadena (no bytes). `upper()` y `lower()` convierten a mayusculas o minusculas respectivamente. `initcap()` pone en mayuscula la primera letra de cada palabra.

```sql
SELECT length('PostgreSQL');          -- 10
SELECT upper('postgres');             -- POSTGRES
SELECT lower('POSTGRES');             -- postgres
SELECT initcap('hola mundo');         -- Hola Mundo
```

`length()` tambien acepta un segundo argumento para indicar la codificacion: `length('ñ', 'UTF8')` sigue devolviendo 1.

### Recorte de espacios

`trim()` elimina espacios (u otros caracteres) del inicio y final. `ltrim()` y `rtrim()` hacen lo mismo solo por la izquierda o la derecha.

```sql
SELECT trim('  hola  ');                   -- 'hola'
SELECT ltrim('  hola  ');                  -- 'hola  '
SELECT rtrim('  hola  ');                  -- '  hola'
SELECT trim(LEADING 'x' FROM 'xxholaxx');  -- 'holaxx'
SELECT trim(BOTH 'x' FROM 'xxholaxx');     -- 'hola'
SELECT trim(TRAILING 'x' FROM 'xxholaxx'); -- 'xxhola'
```

El modificador `LEADING`, `TRAILING` o `BOTH` permite especificar que extremo recortar y que caracter eliminar, no solo espacios.

### Subcadenas y busqueda

`substring()` extrae una porcion de la cadena. `replace()` sustituye todas las ocurrencias de un patron. `position()` localiza la primera aparicion de una subcadena. `split_part()` divide la cadena por un delimitador y devuelve la parte indicada.

```sql
SELECT substring('PostgreSQL', 1, 6);     -- Postgr
SELECT substring('PostgreSQL' FROM 5);    -- SQL
SELECT replace('abc-abc', 'abc', 'xyz');  -- xyz-xyz
SELECT position('SQL' IN 'PostgreSQL');   -- 7
SELECT split_part('a,b,c', ',', 2);       -- b
```

`split_part()` es especialmente util para procesar datos delimitados sin necesidad de recurrir a funciones de parsing externas.

### Concatenacion

El operador `||` concatena dos cadenas. La funcion `concat()` hace lo mismo pero ignorando NULLs, mientras que `concat_ws()` permite especificar un separador.

```sql
SELECT 'Hola' || ' ' || 'Mundo';        -- Hola Mundo
SELECT concat('Hola', NULL, 'Mundo');   -- HolaMundo
SELECT concat_ws(', ', 'a', 'b', 'c');  -- a, b, c
```

La diferencia clave: `||` devuelve NULL si cualquiera de los operandos es NULL, mientras que `concat()` trata los NULLs como cadenas vacias.

### Manipulacion lateral

`left()` y `right()` extraen N caracteres desde el extremo indicado. `repeat()` repite una cadena N veces. `reverse()` invierte el orden de los caracteres.

```sql
SELECT left('PostgreSQL', 4);         -- Post
SELECT right('PostgreSQL', 3);        -- SQL
SELECT repeat('ab', 3);              -- ababab
SELECT reverse('hola');              -- aloh
```

### Funciones de hash y codificacion

`md5()` y `sha256()` calculan el hash de una cadena. Son funciones deterministicas: la misma entrada produce siempre la misma salida. Se usan para almacenar contrasenas con hash o verificar integridad.

```sql
SELECT md5('abc');       -- 900150983cd24fb0d6963f7d28e17f72
SELECT sha256('abc');    -- ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad
```

`encode()` y `decode()` convierten datos binarios (`bytea`) a representaciones de texto como hex o base64, y viceversa.

```sql
SELECT encode('abc'::bytea, 'hex');     -- 616263
SELECT decode('616263', 'hex');         -- \x616263
SELECT encode('abc'::bytea, 'base64');  -- YWJj
SELECT decode('YWJj', 'base64');        -- \x616263
```

> Los hash MD5 y SHA-256 operan sobre texto y devuelven texto. `encode`/`decode` trabajan con el tipo `bytea`, por lo que es necesario hacer casting explicito cuando se parte de una cadena.

## Funciones numericas

Las funciones numericas cubren operaciones aritmeticas, trigonometricas, redondeo y seleccion de valores extremos. Trabajan con los tipos `INTEGER`, `BIGINT`, `NUMERIC` y `DOUBLE PRECISION`, y el tipo del resultado depende del tipo de entrada.

### Redondeo y truncamiento

`ceil()` redondea hacia arriba al entero mas cercano, `floor()` hacia abajo. `round()` redondea al entero mas proximo, con opcion de especificar decimales. `trunc()` trunca sin redondear, tambien con precision configurable.

```sql
SELECT ceil(4.2);         -- 5
SELECT floor(4.9);        -- 4
SELECT round(4.567, 2);   -- 4.57
SELECT trunc(4.567, 2);   -- 4.56
SELECT abs(-17);          -- 17
```

`abs()` devuelve el valor absoluto, comun para calcular distancias o diferencias en valores absolutos.

### Potencia, raiz y modulo

`power()` eleva un numero a una potencia. `sqrt()` calcula la raiz cuadrada. `mod()` devuelve el resto de una division entera.

```sql
SELECT power(2, 10);      -- 1024
SELECT sqrt(144);         -- 12
SELECT mod(10, 3);        -- 1
```

A diferencia del operador `%`, `mod()` maneja correctamente numeros negativos siguiendo la convencion matematica.

### Seleccion de extremos

`greatest()` devuelve el mayor valor de una lista, `least()` el menor. Aunque son funciones numericas, tambien aceptan cualquier tipo comparable.

```sql
SELECT greatest(3, 7, 2, 9);     -- 9
SELECT least(3, 7, 2, 9);        -- 2
SELECT greatest('a', 'z', 'm');  -- z
```

### Aleatoriedad y constantes

`random()` devuelve un valor `DOUBLE PRECISION` entre 0 y 1 (excluyendo 1). `pi()` devuelve la constante π con precision de `DOUBLE PRECISION`.

```sql
SELECT random();                  -- 0.374540...
SELECT random() * 100;           -- Aleatorio entre 0 y 100
SELECT floor(random() * 10 + 1)::int;  -- Entero aleatorio entre 1 y 10
SELECT pi();                      -- 3.14159265358979
```

> `random()` es no determinista: devuelve un valor distinto en cada llamada. No puede usarse en expresiones indexadas ni en funciones deterministicas.

## Funciones de fecha y hora

El manejo de fechas y horas es uno de los puntos fuertes de PostgreSQL. El motor soporta zonas horarias, intervalos, aritmetica de fechas y un ecosistema completo de funciones de extraccion, truncado y formateo.

### Fecha y hora actuales

`now()` y `current_timestamp` devuelven el momento actual como `TIMESTAMPTZ`. `current_date` devuelve solo la fecha, `current_time` solo la hora.

```sql
SELECT now();                       -- 2026-07-28 14:30:00.123456+02
SELECT current_timestamp;           -- 2026-07-28 14:30:00.123456+02
SELECT current_date;                -- 2026-07-28
SELECT current_time;                -- 14:30:00.123456+02
```

`now()` y `current_timestamp` son equivalentes. Dentro de una transaccion, ambas devuelven siempre el mismo valor (el momento de inicio de la transaccion), a menos que se use `clock_timestamp()`.

### Diferencia entre fechas

`age()` calcula la diferencia entre dos fechas expresada en anios, meses y dias. Con un solo argumento, calcula la diferencia respecto a la fecha actual.

```sql
SELECT age('2026-12-25', '2026-07-28');   -- 4 mons 27 days
SELECT age('2026-12-25');                 -- 4 mons 27 days (desde hoy)
```

El resultado es de tipo `INTERVAL`, con campos de anio, mes y dia normalizados.

### Extraccion de componentes

`extract()` obtiene un campo concreto de una fecha, timestamp o intervalo. `date_part()` es funcionalmente identica pero devuelve `DOUBLE PRECISION` en lugar de `NUMERIC`.

```sql
SELECT extract(YEAR FROM TIMESTAMP '2026-07-28 14:30:00');    -- 2026
SELECT extract(MONTH FROM TIMESTAMP '2026-07-28 14:30:00');   -- 7
SELECT extract(DAY FROM TIMESTAMP '2026-07-28 14:30:00');     -- 28
SELECT extract(HOUR FROM TIMESTAMP '2026-07-28 14:30:00');    -- 14
SELECT extract(MINUTE FROM TIMESTAMP '2026-07-28 14:30:00');  -- 30
SELECT extract(SECOND FROM TIMESTAMP '2026-07-28 14:30:00');  -- 0
SELECT extract(DOW FROM TIMESTAMP '2026-07-28');              -- 2 (martes, 0=domingo)
SELECT extract(DOY FROM TIMESTAMP '2026-07-28');              -- 209
SELECT extract(EPOCH FROM TIMESTAMP '2026-07-28');            -- 178... (segundos desde 1970-01-01)

SELECT date_part('year', TIMESTAMP '2026-07-28 14:30:00');    -- 2026
```

Los campos disponibles incluyen `YEAR`, `MONTH`, `DAY`, `HOUR`, `MINUTE`, `SECOND`, `DOW` (dia de la semana), `DOY` (dia del anio), `QUARTER`, `DECADE`, `CENTURY`, `EPOCH`, entre otros.

### Truncado de fechas

`date_trunc()` trunca un timestamp a una precision determinada. Es util para agrupar datos por periodos.

```sql
SELECT date_trunc('year', TIMESTAMP '2026-07-28 14:30:00');    -- 2026-01-01 00:00:00
SELECT date_trunc('month', TIMESTAMP '2026-07-28 14:30:00');   -- 2026-07-01 00:00:00
SELECT date_trunc('day', TIMESTAMP '2026-07-28 14:30:00');     -- 2026-07-28 00:00:00
SELECT date_trunc('hour', TIMESTAMP '2026-07-28 14:30:00');    -- 2026-07-28 14:00:00
SELECT date_trunc('week', TIMESTAMP '2026-07-28 14:30:00');    -- 2026-07-27 00:00:00 (lunes)
```

Combinar `date_trunc` con `GROUP BY` permite generar agregaciones por hora, dia, mes o anio de forma directa.

### Formateo y parseo

`to_char()` convierte una fecha o timestamp a texto con un formato especifico. `to_date()` y `to_timestamp()` convierten texto a fecha o timestamp respectivamente, interpretando el formato indicado.

```sql
SELECT to_char(TIMESTAMP '2026-07-28 14:30:00', 'YYYY-MM-DD HH24:MI:SS');
-- 2026-07-28 14:30:00

SELECT to_char(TIMESTAMP '2026-07-28 14:30:00', 'DD/MM/YYYY');
-- 28/07/2026

SELECT to_char(TIMESTAMP '2026-07-28 14:30:00', 'FMMonth DD, YYYY');
-- July 28, 2026

SELECT to_char(TIMESTAMP '2026-07-28 14:30:00', 'Day, DD "de" Month "de" YYYY');
-- Tuesday  , 28 de July   de 2026

SELECT to_date('28/07/2026', 'DD/MM/YYYY');
-- 2026-07-28

SELECT to_timestamp('28-07-2026 14:30:00', 'DD-MM-YYYY HH24:MI:SS');
-- 2026-07-28 14:30:00+02
```

Los formatos mas comunes para `to_char()` incluyen:

| Patron | Significado              | Ejemplo   |
|--------|--------------------------|-----------|
| YYYY   | Anio en 4 digitos        | 2026      |
| MM     | Mes en 2 digitos         | 07        |
| DD     | Dia en 2 digitos         | 28        |
| HH24   | Hora en formato 24h      | 14        |
| HH12   | Hora en formato 12h      | 02        |
| MI     | Minutos                  | 30        |
| SS     | Segundos                 | 00        |
| MS     | Milisegundos             | 123       |
| US     | Microsegundos            | 123456    |
| Day    | Nombre del dia (capitalizado) | Tuesday  |
| Month  | Nombre del mes           | July      |
| Mon    | Abreviatura del mes      | Jul       |
| AM/PM  | Indicador AM/PM          | PM        |
| TZ     | Zona horaria             | CEST      |
| FX     | Formato exacto (no flexible) | Prefijo |

> `to_char()` anade espacios de relleno para `Day` y `Month`. Para evitarlo se usa el prefijo `FM` (Fill Mode): `FMMonth`, `FMDay`.

### Construccion de fechas

`make_date()`, `make_time()` y `make_timestamp()` construyen valores de fecha, hora o timestamp a partir de componentes numericos individuales.

```sql
SELECT make_date(2026, 7, 28);                         -- 2026-07-28
SELECT make_time(14, 30, 0);                           -- 14:30:00
SELECT make_timestamp(2026, 7, 28, 14, 30, 0);        -- 2026-07-28 14:30:00
SELECT make_timestamptz(2026, 7, 28, 14, 30, 0, 'Europe/Madrid');
-- 2026-07-28 14:30:00+02
```

Son especialmente utiles cuando los componentes de fecha provienen de columnas separadas (anio, mes, dia como enteros).

### Aritmetica con intervalos

Los operadores `+` y `-` permiten sumar o restar intervalos a fechas y timestamps, asi como calcular diferencias entre fechas.

```sql
SELECT TIMESTAMP '2026-07-28 14:30:00' + INTERVAL '1 day';
-- 2026-07-29 14:30:00

SELECT TIMESTAMP '2026-07-28 14:30:00' + INTERVAL '3 months';
-- 2026-10-28 14:30:00

SELECT TIMESTAMP '2026-07-28 14:30:00' - INTERVAL '2 hours';
-- 2026-07-28 12:30:00

SELECT DATE '2026-12-25' - DATE '2026-07-28';
-- 150 (dias enteros)

SELECT TIMESTAMP '2026-12-25 10:00:00' - TIMESTAMP '2026-07-28 14:30:00';
-- 149 days 19:30:00 (tipo interval)
```

Los intervalos pueden combinarse mediante multiplicacion y division:

```sql
SELECT INTERVAL '1 day' * 7;   -- 7 days
SELECT INTERVAL '1 hour' * 24; -- 24:00:00
```

> Multiplicar o sumar intervalos directamente sobre `DATE` promueve el resultado a `TIMESTAMP` si se opera con horas/minutos. Es recomendable hacer casting explicito cuando la precision importa.

