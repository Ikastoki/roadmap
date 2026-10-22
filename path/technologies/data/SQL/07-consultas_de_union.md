<h1 align="center">Consultas de Union (JOIN)</h1>

<h2>📑 Contenido</h2>

- [Consultas de Union (JOIN)](#consultas-de-union-join)
- [INNER JOIN](#inner-join)
- [LEFT JOIN (o LEFT OUTER JOIN)](#left-join-o-left-outer-join)
- [RIGHT JOIN (o RIGHT OUTER JOIN)](#right-join-o-right-outer-join)
- [FULL JOIN (o FULL OUTER JOIN)](#full-join-o-full-outer-join)
- [CROSS JOIN](#cross-join)
- [SELF JOIN](#self-join)

## Consultas de Union (JOIN)

Las "joins queries" (consultas de unión) en SQL son consultas que combinan filas de dos o más tablas en función de una relación entre ellas. Las consultas de unión se utilizan para recuperar datos relacionados almacenados en múltiples tablas de una base de datos.

Cuando necesitas datos de múltiples tablas relacionadas, puedes usar la cláusula JOIN para combinar esas tablas basadas en una condición específica, generalmente una relación entre las columnas de las tablas.

## INNER JOIN

Devuelve solo las filas que tienen una coincidencia en ambas tablas basadas en la condición de unión.

```sql
SELECT *
FROM tabla1
INNER JOIN tabla2 ON tabla1.columna = tabla2.columna;
```

## LEFT JOIN (o LEFT OUTER JOIN)

Devuelve todas las filas de la tabla izquierda y las filas coincidentes de la tabla derecha. Si no hay coincidencias, devuelve NULL para las columnas de la tabla derecha.

```sql
SELECT *
FROM tabla1
LEFT JOIN tabla2 ON tabla1.columna = tabla2.columna;
```

## RIGHT JOIN (o RIGHT OUTER JOIN)

Devuelve todas las filas de la tabla derecha y las filas coincidentes de la tabla izquierda. Si no hay coincidencias, devuelve NULL para las columnas de la tabla izquierda.

```sql
SELECT *
FROM tabla1
RIGHT JOIN tabla2 ON tabla1.columna = tabla2.columna;
```

## FULL JOIN (o FULL OUTER JOIN)

Devuelve todas las filas de ambas tablas y combina las filas cuando hay una coincidencia. Si no hay coincidencias, devuelve NULL para las columnas correspondientes de la tabla que no tiene una coincidencia.

```sql
SELECT *
FROM tabla1
FULL JOIN tabla2 ON tabla1.columna = tabla2.columna;
```

## CROSS JOIN

Devuelve el producto cartesiano de las dos tablas, es decir, todas las combinaciones ## posibles de filas de ambas tablas

```sql
SELECT *
FROM tabla1
CROSS JOIN tabla2;
```

## SELF JOIN

Devuelve un conjunto de resultados que relaciona las filas de una tabla consigo misma. Esto puede ser útil en situaciones donde necesitas comparar o relacionar filas dentro de la misma tabla basándote en ciertas condiciones. La salida de una auto-unión puede variar dependiendo de las condiciones especificadas en la cláusula ON.

```sql
SELECT e1.nombre AS empleado, e2.nombre AS supervisor
FROM empleados e1
INNER JOIN empleados e2 ON e1.supervisor_id = e2.id;
```
