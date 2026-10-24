# Indices parciales, de expresion, multicolumna y unicos

Los indices en PostgreSQL no se limitan a duplicar el contenido de una columna. Es posible indexar solo un subconjunto de filas, aplicar funciones sobre los valores, combinar varias columnas en una sola estructura o forzar la unicidad de los datos. Cada una de estas variantes responde a problemas concretos y, bien utilizadas, reducen el espacio ocupado y mejoran el rendimiento respecto a un indice convencional.

## Indices parciales

Un indice parcial se crea con una clausula `WHERE` que restringe las filas que entran en el indice. Solo las filas que cumplen la condicion se indexan, lo que reduce drasticamente el tamano del indice cuando solo un subconjunto de los datos es relevante para las busquedas.

```sql
CREATE INDEX idx_pedidos_activos ON pedidos(estado) WHERE estado = 'pendiente';
```

Este indice es util si la mayoria de las consultas sobre pedidos filtran por `estado = 'pendiente'`. Las filas con otros estados (entregado, cancelado) simplemente se ignoran en el indice. El planificador de PostgreSQL detecta automaticamente cuando la condicion del `WHERE` de la consulta es compatible con la del indice parcial y lo utiliza.

El beneficio es doble: el indice ocupa menos espacio y las escrituras son mas rapidas porque solo se actualiza cuando las filas afectadas cumplen la condicion. Una fila que pasa de `pendiente` a `entregado` se elimina del indice sin necesidad de tocar el resto.

```sql
-- El planificador usara idx_pedidos_activos
SELECT * FROM pedidos WHERE estado = 'pendiente' AND fecha > '2026-01-01';

-- No usara el indice parcial porque la condicion no coincide
SELECT * FROM pedidos WHERE estado = 'entregado';
```

> La condicion del indice parcial debe involucrar columnas de la tabla y literales, no subconsultas ni referencias a otras tablas.

## Indices de expresion

Un indice de expresion indexa el resultado de aplicar una funcion o expresion aritmetica a una o varias columnas, en lugar de indexar el valor bruto de la columna. Es la solucion cuando las consultas filtran u ordenan por un valor transformado.

```sql
CREATE INDEX idx_usuarios_email_lower ON usuarios(LOWER(email));
```

Sin este indice, una busqueda por `LOWER(email) = 'usuario@example.com'` realizaria un `Seq Scan` completo porque el indice B-Tree sobre `email` almacena los valores originales, no los convertidos a minusculas. Con el indice de expresion, la busqueda es directa.

```sql
-- Usara idx_usuarios_email_lower
SELECT * FROM usuarios WHERE LOWER(email) = 'usuario@example.com';

-- No usara el indice de expresion (la funcion es distinta)
SELECT * FROM usuarios WHERE UPPER(email) = 'USUARIO@EXAMPLE.COM';
```

La expresion debe escribirse exactamente igual en la consulta y en la definicion del indice. PostgreSQL no aplica transformaciones algebraicas para deducir equivalencias, aunque puede usar el indice si la consulta contiene la expresion exacta (incluyendo castings y constantes).

Otros ejemplos comunes:

```sql
CREATE INDEX idx_productos_precio_iva ON productos(precio * 1.21);
CREATE INDEX idx_pedidos_mes ON pedidos(EXTRACT(YEAR FROM fecha), EXTRACT(MONTH FROM fecha));
CREATE INDEX idx_datos_extra ON usuarios((datos->>'nombre'));
```

Los indices de expresion se comportan como indices funcionales en otros motores. Es importante tener en cuenta que la funcion debe ser inmutable (devolver el mismo resultado para la misma entrada siempre); funciones como `random()` o `now()` no pueden indexarse.

## Indices multicolumna

Un indice multicolumna (o compuesto) agrupa varias columnas en una unica estructura B-Tree. El orden de las columnas en la definicion es critico: PostgreSQL puede utilizar el indice para busquedas que involucren un prefijo de las columnas indexadas, de izquierda a derecha.

```sql
CREATE INDEX idx_ordenes_cliente_fecha ON ordenes(cliente_id, fecha);
```

Este indice es efectivo para consultas que filtran por `cliente_id`, o por `cliente_id` y `fecha` juntos. Sin embargo, no ayudara a una consulta que filtre solo por `fecha`, porque `fecha` no es el prefijo izquierdo.

```sql
-- Usa el indice (filtra por la primera columna)
SELECT * FROM ordenes WHERE cliente_id = 123;

-- Usa el indice (filtra por ambas)
SELECT * FROM ordenes WHERE cliente_id = 123 AND fecha > '2026-01-01';

-- NO usa el indice (no incluye la primera columna)
SELECT * FROM ordenes WHERE fecha > '2026-01-01';
```

El orden optimo de las columnas sigue la regla: primero las columnas de igualdad (`=`), luego las de rango (`<`, `>`, `BETWEEN`). Si hay varias columnas de igualdad, primero la de mayor cardinalidad (mas valores distintos), porque reduce el espacio de busqueda mas rapidamente.

PostgreSQL tambien puede realizar un `Index Skip Scan` (desde la version 15) que permite al indice multicolumna saltar entre valores distintos de la primera columna, mejorando consultas que filtran solo por la segunda columna sin necesidad de un indice separado.

```sql
-- Desde PostgreSQL 15 puede usar idx_ordenes_cliente_fecha
-- mediante un Index Skip Scan
SELECT DISTINCT fecha FROM ordenes;
```

## Indices unicos

Un indice unico garantiza que no existan dos filas con el mismo valor (o combinacion de valores) en las columnas indexadas. Se declara con la clausula `UNIQUE` en la creacion del indice.

```sql
CREATE UNIQUE INDEX idx_usuarios_nif ON usuarios(nif);
```

Cualquier intento de insertar o actualizar una fila que genere un valor duplicado lanzara un error. Esto incluye la insercion de NULLs: en PostgreSQL, los NULLs se consideran distintos entre si, por lo que multiples filas con `nif = NULL` son permitidas (a diferencia de otros motores como Oracle o SQL Server).

```sql
INSERT INTO usuarios (nif) VALUES (NULL);  -- OK
INSERT INTO usuarios (nif) VALUES (NULL);  -- OK (los NULLs son distintos)
```

### Indices unicos compuestos

Un indice unico puede abarcar varias columnas, forzando la unicidad de la combinacion:

```sql
CREATE UNIQUE INDEX idx_asignacion_curso_alumno ON matriculas(curso_id, alumno_id);
```

Con este indice, un alumno no puede matricularse dos veces en el mismo curso, pero puede estarlo en cursos distintos. Es el equivalente a una `UNIQUE CONSTRAINT` compuesta.

### Diferencia con UNIQUE CONSTRAINT

Una `UNIQUE CONSTRAINT` crea internamente un indice unico B-Tree sobre las columnas especificadas. La diferencia fundamental no es tecnica en el plano fisico: ambos generan la misma estructura. La diferencia es semantica y de gestion:

- Una `UNIQUE CONSTRAINT` se declara como parte del esquema logico y aparece en `information_schema.table_constraints`. Puede ser referenciada por una `FOREIGN KEY` en otra tabla.
- Un `UNIQUE INDEX` es un objeto de indice independiente, no aparece como restriccion en el catalogo de esquemas y no puede ser destino de una clave foranea.

```sql
-- Como restriccion (visible en information_schema)
ALTER TABLE usuarios ADD CONSTRAINT uq_usuarios_nif UNIQUE (nif);

-- Como indice unico (no visible como restriccion)
CREATE UNIQUE INDEX idx_usuarios_nif ON usuarios(nif);
```

En la practica, si se necesita unicidad con semantica de restriccion (especialmente para claves foraneas), se usa `UNIQUE CONSTRAINT`. Si solo se busca unicidad con control directo sobre el tipo de indice o propiedades como `WHERE` (indice parcial unico), se usa `CREATE UNIQUE INDEX`.

```sql
-- Indice unico parcial: solo garantiza unicidad en pedidos pendientes
CREATE UNIQUE INDEX idx_pedido_unico_pendiente ON pedidos(numero) WHERE estado = 'pendiente';
```

Esto permite que dos pedidos tengan el mismo numero si uno esta pendiente y otro cancelado, pero no si ambos estan pendientes. Una `UNIQUE CONSTRAINT` no permite la clausula `WHERE`.

