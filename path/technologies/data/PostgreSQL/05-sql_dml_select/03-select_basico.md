# SELECT basico y filtrado

SELECT es la sentencia fundamental de consulta en SQL. Su sintaxis, aparentemente sencilla, esconde una maquinaria de procesamiento que involucra al optimizador de consultas, el planificador y el motor de ejecucion. Comprender cada clausula y el orden logico en que se evaluan es el primer paso para escribir consultas correctas y eficientes.

## Sintaxis basica

La estructura minima es `SELECT ... FROM ...`. El motor evalua primero la clausula FROM para determinar las filas de entrada, y despues SELECT para proyectar las columnas deseadas:

```sql
SELECT * FROM clientes;
SELECT id, nombre, email FROM clientes;
```

El comodin `*` selecciona todas las columnas en el orden definido en la tabla. Aunque comodo, es preferible enumerar las columnas explicitamente en codigo de produccion para evitar sorpresas si la estructura de la tabla cambia.

### AS: alias de columnas

Los alias hacen que los resultados sean mas legibles y son obligatorios cuando se usan expresiones o funciones de agregacion:

```sql
SELECT
    id,
    nombre AS cliente,
    email,
    total * 1.21 AS total_con_iva
FROM pedidos;
```

La palabra clave `AS` es opcional: `SELECT nombre cliente FROM clientes` funciona igual, pero incluirla mejora la legibilidad.

## WHERE

WHERE filtra las filas antes de que lleguen a SELECT. Solo las filas que evaluan la condicion como `TRUE` pasan al siguiente paso del procesamiento. Las filas que evaluan `FALSE` o `NULL` son descartadas:

```sql
SELECT * FROM productos
WHERE precio > 100;
```

### Operadores de comparacion

```sql
-- igualdad y desigualdad
WHERE email = 'ana@example.com';
WHERE precio <> 0;
WHERE estado != 'cancelado';   -- <> es estandar, != es extension

-- comparacion de orden
WHERE precio < 50;
WHERE stock >= 10;

-- BETWEEN (rango inclusivo en ambos extremos)
WHERE fecha BETWEEN '2025-01-01' AND '2025-12-31';

-- IN (pertenencia a conjunto)
WHERE estado IN ('pendiente', 'procesando', 'enviado');
WHERE id NOT IN (SELECT cliente_id FROM morosos);

-- LIKE e ILIKE (patrones de texto)
WHERE nombre LIKE 'Ana%';       -- sensible a mayusculas
WHERE email ILIKE '%@gmail.com'; -- insensible a mayusculas

-- IS NULL (comparacion con NULL)
WHERE fecha_baja IS NULL;
WHERE fecha_baja IS NOT NULL;
```

> NULL no es un valor, sino la ausencia de el. Cualquier comparacion con NULL usando los operadores habituales (=, <>, <) produce NULL, que en WHERE se interpreta como falso. Por eso existe `IS NULL`.

### LIKE e ILIKE

Los patrones LIKE usan dos comodines: `%` (cualquier secuencia de caracteres) y `_` (un solo caracter):

```sql
WHERE nombre LIKE 'A%';    -- empieza por A
WHERE nombre LIKE '%ez';   -- termina en "ez"
WHERE nombre LIKE '___';   -- exactamente 3 caracteres
WHERE nombre LIKE 'A_%';   -- empieza por A, al menos 2 caracteres
```

`ILIKE` es la version insensible a mayusculas, especifica de PostgreSQL. En el estandar SQL, la insensibilidad se logra con `LOWER()`.

## ORDER BY

ORDER BY ordena el resultado final de la consulta segun una o mas columnas o expresiones. Es la ultima clausula en ejecutarse (despues de SELECT), por lo que puede referirse a alias definidos en SELECT:

```sql
SELECT nombre, salario / 12 AS salario_mensual
FROM empleados
ORDER BY salario_mensual DESC;
```

### ASC, DESC y nulos

```sql
ORDER BY fecha ASC;          -- ascendente (por defecto)
ORDER BY total DESC;         -- descendente
ORDER BY departamento ASC, salario DESC;  -- orden compuesto
ORDER BY nombre NULLS LAST;  -- nulos al final
ORDER BY nombre NULLS FIRST; -- nulos al principio
```

Por defecto, `ASC` coloca los NULL al final y `DESC` al principio. Las clausulas `NULLS FIRST` y `NULLS LAST` permiten controlar este comportamiento explicitamente.

## LIMIT y OFFSET

LIMIT restringe el numero de filas devueltas; OFFSET salta un numero de filas antes de empezar a devolver resultados:

```sql
SELECT * FROM productos
ORDER BY precio DESC
LIMIT 10;
```

```sql
SELECT * FROM productos
ORDER BY precio DESC
LIMIT 10 OFFSET 20;
```

> OFFSET sin ORDER BY no tiene sentido logico: el orden de las filas en SQL no esta garantizado a menos que se especifique explicitamente. Sin ORDER BY, el mismo OFFSET puede devolver filas distintas en cada ejecucion.

## DISTINCT

DISTINCT elimina filas duplicadas del resultado. Opera sobre todas las columnas seleccionadas:

```sql
SELECT DISTINCT ciudad FROM clientes;
SELECT DISTINCT departamento, puesto FROM empleados;
```

`DISTINCT ON` es una extension de PostgreSQL que permite mantener solo la primera fila de cada grupo definido por una o mas columnas:

```sql
SELECT DISTINCT ON (cliente_id)
    cliente_id,
    fecha,
    total
FROM pedidos
ORDER BY cliente_id, fecha DESC;
```

Devuelve el pedido mas reciente de cada cliente. Las columnas dentro de `DISTINCT ON` deben coincidir con el inicio de `ORDER BY` para que el resultado sea deterministico.

## Operadores logicos

### AND, OR, NOT

Combinan condiciones booleanas siguiendo las reglas del algebra de Boole con tres valores (TRUE, FALSE, NULL):

```sql
WHERE activo = true AND saldo > 0;
WHERE departamento = 'ventas' OR departamento = 'soporte';
WHERE NOT estado = 'cancelado';
WHERE (departamento = 'ventas' OR departamento = 'soporte')
  AND activo = true;
```

El cortocircuito logico se aplica: en `A AND B`, si A es FALSE, B no se evalua. En `A OR B`, si A es TRUE, B no se evalua.

### EXISTS

EXISTS evalua si una subconsulta devuelve al menos una fila. Es un predicado semantico, no necesita el contenido de las filas, solo su existencia:

```sql
SELECT nombre, email
FROM clientes c
WHERE EXISTS (
    SELECT 1 FROM pedidos p
    WHERE p.cliente_id = c.id
      AND p.total > 1000
);
```

La subconsulta suele escribirse con `SELECT 1` porque el contenido es irrelevante: EXISTS solo comprueba si hay filas.

### ANY y ALL

ANY compara un valor con cada elemento de un conjunto devuelto por una subconsulta. Si alguna comparacion es TRUE, ANY es TRUE:

```sql
SELECT nombre
FROM productos
WHERE id = ANY (SELECT producto_id FROM pedidos_detalle WHERE cantidad > 10);
```

ALL requiere que todas las comparaciones sean TRUE:

```sql
SELECT nombre, precio
FROM productos
WHERE precio > ALL (SELECT precio FROM productos WHERE categoria = 'basico');
```

ANY y ALL pueden combinarse con operadores de comparacion: `= ANY(...)` equivale a `IN(...)`, pero `<> ALL(...)` equivale a `NOT IN(...)`.

