# JOINs y subconsultas

Las bases de datos relacionales almacenan la informacion distribuida en multiples tablas vinculadas mediante claves. JOIN es el mecanismo que reconstruye la vista completa combinando filas de distintas tablas segun una condicion de correspondencia. Las subconsultas, por su parte, permiten incrustar consultas dentro de otras consultas, ofreciendo una via alternativa para relacionar datos sin necesidad de JOINs explicitos.

## JOINs

### INNER JOIN

Devuelve solo las filas donde existe correspondencia en ambas tablas. Es el JOIN mas utilizado y, en ausencia de especificacion contraria, el que mejor rendimiento ofrece porque descarta filas sin pareja:

```sql
SELECT
    p.id AS pedido_id,
    c.nombre AS cliente,
    p.total,
    p.fecha
FROM pedidos p
INNER JOIN clientes c ON p.cliente_id = c.id;
```

### LEFT JOIN (y LEFT JOIN exclusivo)

LEFT JOIN conserva todas las filas de la tabla izquierda (la del FROM) y las correspondientes de la derecha. Cuando no hay correspondencia, las columnas de la derecha aparecen como NULL:

```sql
SELECT
    c.id,
    c.nombre,
    COUNT(p.id) AS total_pedidos
FROM clientes c
LEFT JOIN pedidos p ON p.cliente_id = c.id
GROUP BY c.id, c.nombre;
```

Para obtener solo las filas de la izquierda que no tienen correspondencia (LEFT JOIN exclusivo), se filtra con `WHERE columna_derecha IS NULL`:

```sql
SELECT c.nombre, c.email
FROM clientes c
LEFT JOIN pedidos p ON p.cliente_id = c.id
WHERE p.id IS NULL;
```

Devuelve los clientes que nunca han hecho un pedido.

### RIGHT JOIN

Equivalente a LEFT JOIN pero con la tabla derecha como la que conserva todas las filas. En la practica es poco usado porque se prefiere dar la vuelta al orden de las tablas y usar LEFT JOIN:

```sql
SELECT c.nombre, p.id
FROM pedidos p
RIGHT JOIN clientes c ON p.cliente_id = c.id;
```

Es semanticamente identico a `FROM clientes c LEFT JOIN pedidos p ...`.

### FULL JOIN

Combina LEFT y RIGHT: conserva todas las filas de ambas tablas. Las filas sin correspondencia muestran NULL en la tabla opuesta:

```sql
SELECT
    COALESCE(c.nombre, '(sin cliente)') AS cliente,
    COALESCE(p.id::TEXT, '(sin pedido)') AS pedido
FROM clientes c
FULL JOIN pedidos p ON p.cliente_id = c.id;
```

### CROSS JOIN

Producto cartesiano: cada fila de la tabla A se combina con cada fila de la tabla B. Sin WHERE, el numero de filas resultante es |A| x |B|:

```sql
SELECT tallas.talla, colores.color
FROM tallas
CROSS JOIN colores;
```

Es util para generar combinaciones completas, como todas las variantes posibles de un producto.

### NATURAL JOIN

JOIN implicito que empareja columnas del mismo nombre en ambas tablas. Aunque reduce la escritura, es peligroso en produccion porque cualquier cambio en los nombres de columna altera silenciosamente la logica del JOIN:

```sql
SELECT * FROM pedidos NATURAL JOIN clientes;
```

Equivale a `INNER JOIN clientes USING (id)` si ambas tablas tienen una columna llamada `id`. No se recomienda su uso en codigo que deba mantenerse a largo plazo.

### SELF JOIN

Una tabla se combina consigo misma. Requiere alias para distinguir las dos instancias:

```sql
SELECT
    e.nombre AS empleado,
    j.nombre AS jefe
FROM empleados e
LEFT JOIN empleados j ON e.jefe_id = j.id;
```

Es la forma de representar relaciones jerarquicas (arbol, organigrama) en una base de datos relacional.

### LATERAL JOIN

LATERAL permite que una subconsulta en el FROM haga referencia a columnas de tablas anteriores en la misma clausula FROM. Es una caracteristica avanzada que resuelve problemas que de otro modo requerian subconsultas correlacionadas o CTEs:

```sql
SELECT
    c.nombre,
    ultimos_pedidos.*
FROM clientes c
LEFT JOIN LATERAL (
    SELECT p.id, p.total, p.fecha
    FROM pedidos p
    WHERE p.cliente_id = c.id
    ORDER BY p.fecha DESC
    LIMIT 3
) ultimos_pedidos ON true;
```

La subconsulta se ejecuta para cada fila de `clientes`, devolviendo los tres pedidos mas recientes de cada uno. Sin LATERAL, la subconsulta no podria referenciar `c.id`.

### USING vs ON

USING es azucar sintactico para JOINs donde las columnas de enlace tienen el mismo nombre en ambas tablas:

```sql
SELECT * FROM pedidos INNER JOIN clientes USING (cliente_id);

-- equivalente con ON
SELECT * FROM pedidos INNER JOIN clientes ON pedidos.cliente_id = clientes.cliente_id;
```

USING tiene la ventaja adicional de que la columna de enlace aparece una sola vez en el resultado, mientras que con ON aparece duplicada. Sin embargo, ON es mas flexible porque permite condiciones arbitrarias, no solo igualdad de nombres.

## Subconsultas

### Subconsultas en FROM (derived tables)

Una subconsulta en FROM actua como una tabla virtual. Debe tener un alias y no puede acceder a columnas de la consulta exterior (a menos que sea LATERAL):

```sql
SELECT
    d.nombre,
    d.total_ventas
FROM (
    SELECT
        c.nombre,
        SUM(p.total) AS total_ventas
    FROM clientes c
    JOIN pedidos p ON p.cliente_id = c.id
    GROUP BY c.id, c.nombre
    HAVING SUM(p.total) > 500
) d
ORDER BY d.total_ventas DESC;
```

### Subconsultas correlacionadas

Una subconsulta correlacionada referencia columnas de la consulta externa y se reevalua para cada fila de esta. Pueden ser ineficientes con tablas grandes si no hay indices adecuados:

```sql
SELECT
    c.nombre,
    c.email
FROM clientes c
WHERE (
    SELECT COUNT(*)
    FROM pedidos p
    WHERE p.cliente_id = c.id
) > 5;
```

Devuelve clientes con mas de 5 pedidos. La subconsulta se ejecuta una vez por cada cliente.

### Subconsultas en JOINs

Una subconsulta puede aparecer en la clausula JOIN, funcionando como una tabla derivada. Es especialmente util para precalcular agregados antes de la combinacion:

```sql
SELECT
    c.nombre,
    c.email,
    resumen.total_pedidos,
    resumen.gasto_total
FROM clientes c
JOIN (
    SELECT
        cliente_id,
        COUNT(*) AS total_pedidos,
        SUM(total) AS gasto_total
    FROM pedidos
    GROUP BY cliente_id
) resumen ON resumen.cliente_id = c.id;
```

Este patron equivale a una agregacion con GROUP BY en la consulta principal, pero separa la responsabilidad: la subconsulta calcula los agregados y el JOIN exterior los combina con los datos del cliente.

### Subconsultas escalares

Una subconsulta que devuelve una unica fila y columna puede usarse como expresion en SELECT o WHERE:

```sql
SELECT
    nombre,
    salario,
    (SELECT AVG(salario) FROM empleados) AS media_empresa,
    salario - (SELECT AVG(salario) FROM empleados) AS diferencia
FROM empleados;
```

> Las subconsultas escalares que devuelven mas de una fila producen un error en tiempo de ejecucion. Si devuelven cero filas, el resultado es NULL.

