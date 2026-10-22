<h1 align="center">Consultas Agregadas</h1>

<h2>📑 Contenido</h2>

- [Consultas Agregadas](#consultas-agregadas)
- [SUM](#sum)
- [COUNT](#count)
- [AVG](#avg)
- [MAX](#max)
- [MIN](#min)
- [GROUP BY](#group-by)
- [HAVING](#having)

## Consultas Agregadas

Las "aggregated queries" o consultas agregadas son consultas en bases de datos que utilizan funciones de agregación, como SUM, COUNT, AVG, MAX o MIN, para combinar múltiples filas de datos en un conjunto de resultados único. Estas consultas se utilizan comúnmente para resumir datos y obtener información estadística de grandes conjuntos de datos. Por ejemplo, podrías usar una consulta agregada para encontrar el total de ventas de un producto, el promedio de edad de los clientes, o el número de registros que cumplen ciertos criterios.

## SUM

La función SUM es una función de agregación comúnmente utilizada en consultas de bases de datos para calcular la suma de los valores en una columna específica. Por ejemplo, si tienes una tabla de ventas con una columna que registra el monto de cada venta, puedes usar la función SUM para obtener la suma total de todas esas ventas.

```sql
SELECT SUM(columna) FROM tabla;
```

## COUNT

La función COUNT es otra función de agregación comúnmente utilizada en consultas de bases de datos para contar el número de filas que cumplen ciertos criterios. Por ejemplo, puedes contar el número de registros en una tabla que cumplen con ciertas condiciones o contar el número total de registros en una tabla.

```sql
SELECT COUNT(*) FROM tabla WHERE edad > 18;
```

## AVG

La función AVG (promedio) es otra función de agregación comúnmente utilizada en consultas de bases de datos. Esta función calcula el promedio de los valores en una columna específica de una tabla.

```sql
SELECT AVG(monto) FROM ventas;
```

## MAX

La función MAX es una función de agregación que se utiliza para encontrar el valor máximo en una columna específica de una tabla en una base de datos. Esta función devuelve el valor máximo encontrado en la columna especificada.

```sql
SELECT MAX(monto) FROM ventas;
```

## MIN

La función MIN es otra función de agregación utilizada en consultas de bases de datos para encontrar el valor mínimo en una columna específica de una tabla. Esta función devuelve el valor mínimo encontrado en la columna especificada.

```sql
SELECT MIN(precio) FROM productos;
```

## GROUP BY

La cláusula GROUP BY se utiliza en consultas SQL junto con funciones de agregación (como SUM, COUNT, AVG, MAX, MIN) para agrupar filas que tienen el mismo valor en una o más columnas y luego aplicar una función de agregación a cada grupo.

```sql
SELECT producto, SUM(cantidad) AS total_vendido
FROM ventas
GROUP BY producto;
```

## HAVING

La cláusula HAVING se utiliza en combinación con la cláusula GROUP BY en consultas SQL para filtrar grupos de filas que resultan de la agrupación, basándose en una condición específica.

```sql
SELECT producto, SUM(cantidad) AS total_vendido
FROM ventas
GROUP BY producto
HAVING SUM(cantidad) > 100;
```
