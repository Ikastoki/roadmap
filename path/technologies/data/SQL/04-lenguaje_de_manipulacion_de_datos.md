<h1 align="center">Lenguaje de manipulación de datos (DML)</h1>

<h2>📑 Contenido</h2>

- [Lenguaje de manipulación de datos (DML)](#lenguaje-de-manipulación-de-datos-dml)
- [SELECT](#select)
- [SELECT DISTINCT](#select-distinct)
- [ORDER BY](#order-by)
- [GROUP BY](#group-by)
- [HAVING](#having)
- [JOINS](#joins)
- [INSERT INTO](#insert-into)
- [UPDATE](#update)
- [DELETE FROM](#delete-from)

## Lenguaje de manipulación de datos (DML)

El Lenguaje de Manipulación de Datos (DML, por sus siglas en inglés) en SQL se utiliza para interactuar con los datos almacenados en una base de datos.

## SELECT

Recupera datos de una o más tablas.

```sql
SELECT columna1, columna2 FROM nombre_de_la_tabla WHERE condición;
```

## SELECT DISTINCT

Se utiliza junto con la cláusula SELECT para seleccionar valores únicos de una columna. Elimina duplicados de los resultados de la consulta.

```sql
SELECT DISTINCT columna FROM tabla;
```

## ORDER BY

Se utiliza para ordenar los resultados de una consulta basándose en el valor de una o más columnas. Puede ordenarse de forma ascendente (ASC) o descendente (DESC), siendo ASC la opción predeterminada.

```sql
SELECT columna1, columna2 FROM tabla ORDER BY columna1 ASC;
```

## GROUP BY

Se utiliza para agrupar filas que tienen el mismo valor en una o más columnas, y se utiliza en conjunto con funciones de agregación (como SUM, COUNT, AVG, etc.) para resumir los datos.

```sql
SELECT columna1, COUNT(*) FROM tabla GROUP BY columna1;
```

## HAVING

Se utiliza junto con la cláusula GROUP BY para filtrar los resultados de una consulta basándose en una condición de agregación. Funciona de manera similar a la cláusula WHERE, pero se aplica después de la agrupación.

```sql
SELECT columna1, COUNT(*) FROM tabla GROUP BY columna1 HAVING COUNT(*) > 1;
```

## JOINS

Se utilizan para combinar datos de dos o más tablas basándose en una condición de relación entre ellas. Hay varios tipos de JOIN, incluyendo INNER JOIN, LEFT JOIN, RIGHT JOIN, y FULL OUTER JOIN.

```sql
SELECT tabla1.columna1, tabla2.columna2 FROM tabla1 JOIN tabla2 ON tabla1.columna_id = tabla2.columna_id;
```

## INSERT INTO

Inserta nuevos registros en una tabla.

```sql
INSERT INTO nombre_de_la_tabla (columna1, columna2) VALUES (valor1, valor2);
```

## UPDATE

Actualiza datos existentes en una tabla.

```sql
UPDATE nombre_de_la_tabla SET columna1 = valor1 WHERE condición;
```

## DELETE FROM

Elimina registros de una tabla.

```sql
DELETE FROM nombre_de_la_tabla WHERE condición;
```
