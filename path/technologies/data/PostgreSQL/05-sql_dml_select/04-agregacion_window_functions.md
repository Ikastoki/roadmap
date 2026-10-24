# Agregacion y funciones ventana

Las consultas agregadas colapsan multiples filas en un unico resultado mediante funciones que resumen, totalizan o promedian. Las funciones ventana, por el contrario, mantienen la granularidad de las filas originales mientras calculan valores en un contexto definido por particiones y orden. Ambas herramientas son esenciales para el analisis de datos dentro de la propia base de datos.

## Agregacion

### COUNT, SUM, AVG, MIN, MAX

Las funciones de agregacion operan sobre un conjunto de filas y producen un unico valor por grupo:

```sql
SELECT
    COUNT(*) AS total_filas,
    COUNT(email) AS emails_no_nulos,
    COUNT(DISTINCT ciudad) AS ciudades_distintas,
    SUM(total) AS suma_total,
    AVG(total) AS promedio,
    MIN(fecha) AS primer_pedido,
    MAX(fecha) AS ultimo_pedido
FROM pedidos;
```

Algunos matices importantes:

- `COUNT(*)` cuenta filas independientemente de valores NULL.
- `COUNT(columna)` cuenta solo filas donde la columna no es NULL.
- `SUM` y `AVG` ignoran NULLs. Si todas las filas son NULL, devuelven NULL.
- `MIN` y `MAX` funcionan con cualquier tipo de datos comparable (numeros, fechas, texto).

### GROUP BY

GROUP BY divide las filas en grupos segun los valores de una o mas columnas. Las funciones de agregacion se aplican entonces a cada grupo de forma independiente:

```sql
SELECT
    departamento,
    COUNT(*) AS empleados,
    AVG(salario)::NUMERIC(10,2) AS salario_promedio,
    MAX(salario) - MIN(salario) AS rango_salarial
FROM empleados
GROUP BY departamento;
```

Todas las columnas del SELECT que no sean funciones de agregacion deben aparecer en GROUP BY. PostgreSQL es mas permisivo que otros motores si la columna omitida es funcionalmente dependiente de las agrupadas (por ejemplo, `nombre` cuando se agrupa por `id`), pero confiar en esta extension puede producir resultados ambiguos.

```sql
-- agrupacion por multiples columnas
SELECT
    departamento,
    puesto,
    COUNT(*) AS plantilla
FROM empleados
GROUP BY departamento, puesto
ORDER BY departamento, plantilla DESC;
```

### HAVING

HAVING filtra grupos despues de la agregacion, de forma analoga a como WHERE filtra filas antes de ella. La diferencia fundamental es que HAVING puede hacer referencia a funciones de agregacion, mientras que WHERE no:

```sql
SELECT
    cliente_id,
    COUNT(*) AS num_pedidos,
    SUM(total) AS gasto_total
FROM pedidos
WHERE estado != 'cancelado'
GROUP BY cliente_id
HAVING COUNT(*) >= 5 AND SUM(total) > 1000
ORDER BY gasto_total DESC;
```

El orden logico de ejecucion es: FROM -> WHERE -> GROUP BY -> HAVING -> SELECT -> ORDER BY. Por eso WHERE no puede usar alias de SELECT ni funciones de agregacion, y HAVING si puede.

## Window functions

Las funciones ventana realizan calculos sobre un conjunto de filas relacionadas con la fila actual, pero sin colapsar el resultado. Cada fila conserva su identidad y ademas recibe el valor calculado.

### OVER() basico

La clausula OVER define la ventana. Sin ninguna especificacion, la ventana comprende todas las filas del resultado:

```sql
SELECT
    nombre,
    salario,
    AVG(salario) OVER() AS salario_promedio_global,
    salario - AVG(salario) OVER() AS diferencia
FROM empleados;
```

### ROW_NUMBER, RANK, DENSE_RANK

Estas funciones de numeracion asignan una posicion a cada fila dentro de la ventana, normalmente en combinacion con ORDER BY:

```sql
SELECT
    nombre,
    salario,
    departamento,
    ROW_NUMBER() OVER (ORDER BY salario DESC) AS posicion_global,
    RANK()       OVER (ORDER BY salario DESC) AS ranking_con_empates,
    DENSE_RANK() OVER (ORDER BY salario DESC) AS ranking_sin_saltos
FROM empleados;
```

La diferencia entre las tres se manifiesta cuando hay empates:

- `ROW_NUMBER`: asigna numeros unicos consecutivos. Los empates se deshacen arbitrariamente (o segun el orden adicional si se especifica).
- `RANK`: asigna el mismo numero a filas empatadas, pero deja saltos. Tras dos filas en el puesto 1, la siguiente es la 3.
- `DENSE_RANK`: asigna el mismo numero a filas empatadas y no deja saltos. Tras dos filas en el puesto 1, la siguiente es la 2.

### NTILE

Divide las filas en un numero especificado de grupos (bucket) lo mas equilibrados posible:

```sql
SELECT
    nombre,
    salario,
    NTILE(4) OVER (ORDER BY salario DESC) AS cuartil
FROM empleados;
```

`NTILE(4)` crea cuatro grupos. Si las filas no se dividen exactamente, los grupos mas grandes son los primeros.

### LAG y LEAD

Acceden a filas anteriores (LAG) o posteriores (LEAD) dentro de la ventana sin necesidad de self-joins:

```sql
SELECT
    fecha,
    total,
    LAG(total)  OVER (ORDER BY fecha) AS total_dia_anterior,
    LEAD(total) OVER (ORDER BY fecha) AS total_dia_siguiente,
    total - LAG(total) OVER (ORDER BY fecha) AS variacion_diaria
FROM ventas_diarias;
```

Ambas funciones aceptan un segundo parametro para el desplazamiento (por defecto 1) y un tercero para el valor por defecto cuando no hay fila anterior/siguiente:

```sql
LAG(total, 7, 0) OVER (ORDER BY fecha) AS total_hace_una_semana
```

### FIRST_VALUE y LAST_VALUE

Devuelven el valor de la primera o ultima fila de la ventana:

```sql
SELECT
    fecha,
    total,
    FIRST_VALUE(total) OVER (ORDER BY fecha) AS primer_valor,
    LAST_VALUE(total)  OVER (ORDER BY fecha ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS ultimo_valor
FROM ventas_diarias;
```

> `LAST_VALUE` requiere una especificacion explicita del marco de ventana (`ROWS BETWEEN ...`) porque por defecto el marco termina en la fila actual, no al final de la particion. Sin ella, `LAST_VALUE` devolveria la fila actual misma.

### PARTITION BY

PARTITION BY divide las filas en particiones independientes. La funcion ventana se reinicia en cada particion:

```sql
SELECT
    nombre,
    departamento,
    salario,
    ROW_NUMBER() OVER (PARTITION BY departamento ORDER BY salario DESC) AS posicion_en_departamento,
    AVG(salario) OVER (PARTITION BY departamento) AS media_departamento
FROM empleados;
```

Cada departamento tiene su propia numeracion y su propia media. Sin PARTITION BY, los calculos serian globales.

### ORDER BY en ventanas

Dentro de OVER, ORDER BY define el orden logico de las filas y, en funciones como SUM o AVG, convierte el calculo en acumulativo (running total):

```sql
SELECT
    fecha,
    total,
    SUM(total) OVER (ORDER BY fecha) AS acumulado,
    AVG(total) OVER (ORDER BY fecha ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS media_movil_7dias
FROM ventas_diarias
ORDER BY fecha;
```

### ROWS BETWEEN y RANGE BETWEEN

El marco de ventana (frame) define que filas incluir respecto a la fila actual. Las opciones principales:

```sql
-- todas las filas hasta la actual (por defecto si hay ORDER BY)
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW

-- media movil de 7 dias (6 anteriores + actual)
ROWS BETWEEN 6 PRECEDING AND CURRENT ROW

-- todas las filas de la particion
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING

-- solo la fila actual
ROWS BETWEEN CURRENT ROW AND CURRENT ROW

-- basado en rango de valores (en lugar de numero de filas)
RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW
```

`ROWS` usa un recuento fisico de filas. `RANGE` usa un intervalo logico basado en el valor de la columna de ordenacion, lo que permite incluir filas cuyos valores de ordenacion esten dentro de un rango, independientemente de cuantas sean.

