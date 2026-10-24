# DML: insercion, actualizacion y borrado de datos

Las sentencias DML (Data Manipulation Language) son el mecanismo mediante el cual los datos fluyen hacia y desde las tablas. INSERT, UPDATE y DELETE constituyen el trio fundamental que, junto con SELECT, forma el nucleo operativo de cualquier aplicacion que persista informacion en una base de datos relacional. PostgreSQL extiende el estandar SQL con capacidades como RETURNING y ON CONFLICT que transforman operaciones triviales en herramientas de alta precision.

## INSERT

### INSERT con valores literales

La forma mas basica de INSERT especifica una lista de columnas y sus valores correspondientes. Cuando se omiten las columnas, se asume el orden definido en CREATE TABLE.

```sql
INSERT INTO usuarios (nombre, email, activo) VALUES
    ('Ana Lopez', 'ana@example.com', true);

INSERT INTO usuarios VALUES
    (DEFAULT, 'Luis Ruiz', 'luis@example.com', true);
```

Es posible insertar multiples filas en una sola sentencia, lo que resulta drasticamente mas eficiente que INSERT individuales gracias a que se reduce el numero de viajes al servidor y se agrupa el log WAL:

```sql
INSERT INTO pedidos (cliente_id, total, estado)
VALUES
    (1, 150.00, 'pendiente'),
    (2, 230.50, 'completado'),
    (3, 89.99, 'pendiente');
```

### INSERT desde SELECT

La variante mas potente de INSERT toma datos directamente del resultado de una consulta SELECT. Esto permite poblar tablas desde otras tablas, realizar transformaciones en el vuelo y cargar datos de sistemas externos:

```sql
INSERT INTO clientes_vip (nombre, email, total_gastado)
SELECT
    c.nombre,
    c.email,
    SUM(p.total)
FROM clientes c
JOIN pedidos p ON p.cliente_id = c.id
GROUP BY c.id, c.nombre, c.email
HAVING SUM(p.total) > 1000;
```

El SELECT puede incluir cualquier construccion del lenguaje: JOINs, agregaciones, funciones de ventana, CTEs e incluso ORDER BY y LIMIT cuando tenga sentido logico.

### UPSERT con ON CONFLICT

PostgreSQL implementa la operacion upsert (insertar o actualizar si existe conflicto) mediante la clausula ON CONFLICT. Es una de las caracteristicas DML mas valoradas porque elimina la necesidad de consultas previas para decidir entre INSERT y UPDATE, resolviendo atomicamente la decision en una sola sentencia:

```sql
INSERT INTO inventario (producto_id, cantidad, almacen)
VALUES (42, 10, 'principal')
ON CONFLICT (producto_id, almacen)
DO UPDATE SET
    cantidad = inventario.cantidad + EXCLUDED.cantidad,
    updated_at = now();
```

La palabra clave EXCLUDED hace referencia a los valores que se estaban intentando insertar. Sin ella, no habria forma de acceder a la nueva fila dentro de la rama DO UPDATE. El conflicto se puede detectar sobre una constraint unica o una exclusion constraint.

#### DO NOTHING

Cuando el conflicto no requiere actualizacion, se omite la fila silenciosamente:

```sql
INSERT INTO logs (evento, origen, hash)
VALUES ('login_ok', '192.168.1.10', 'abc123')
ON CONFLICT (hash) DO NOTHING;
```

#### DO UPDATE con condiciones

Es posible afinar el UPDATE con una clausula WHERE adicional para decidir si realmente se actualiza, evitando writes innecesarios:

```sql
INSERT INTO metricas (servicio, momento, valor)
VALUES ('api-gateway', now(), 0.95)
ON CONFLICT (servicio, momento)
DO UPDATE SET
    valor = EXCLUDED.valor
WHERE metricas.valor IS DISTINCT FROM EXCLUDED.valor;
```

## UPDATE

### UPDATE basico y condicional

UPDATE modifica filas existentes. La clausula WHERE determina que filas se ven afectadas; sin ella, todas las filas de la tabla se actualizan, lo que rara vez es deseable:

```sql
UPDATE usuarios
SET ultimo_acceso = now()
WHERE id = 42;

UPDATE pedidos
SET estado = 'cancelado', updated_at = now()
WHERE cliente_id = 99 AND estado = 'pendiente';
```

### UPDATE con CASE

CASE dentro de SET permite aplicar logica condicional por fila en una sola sentencia, evitando multiples UPDATEs:

```sql
UPDATE pedidos
SET estado = CASE
    WHEN total < 50 THEN 'revision'
    WHEN total >= 50 AND total < 200 THEN 'confirmado'
    WHEN total >= 200 THEN 'prioritario'
    ELSE estado
END
WHERE fecha >= '2025-01-01';
```

### UPDATE con JOIN

PostgreSQL permite actualizar una tabla basandose en valores de otra mediante la sintaxis FROM ... JOIN. Esto es especialmente util para sincronizar tablas o propagar cambios:

```sql
UPDATE productos p
SET stock = stock + i.cantidad
FROM inventario_entrada i
WHERE p.id = i.producto_id
  AND i.procesado = false;
```

La sintaxis USING es equivalente y mas concisa en algunos dialectos, aunque PostgreSQL prefiere FROM en UPDATE.

## DELETE

### DELETE basico

DELETE elimina filas de una tabla. La clausula WHERE es opcional pero peligrosa sin ella: DELETE FROM tabla sin WHERE vacia la tabla por completo (aunque TRUNCATE es mas rapido para ese proposito):

```sql
DELETE FROM sesiones_expiradas
WHERE ultimo_acceso < now() - INTERVAL '30 days';
```

### DELETE con subconsultas

Cuando la condicion de borrado depende de datos en otras tablas, las subconsultas proporcionan la expresividad necesaria:

```sql
DELETE FROM carritos_abandonados c
WHERE NOT EXISTS (
    SELECT 1 FROM pedidos p
    WHERE p.carrito_id = c.id
)
AND c.creado < now() - INTERVAL '7 days';
```

Tambien es posible usar USING para unir tablas directamente en DELETE, similar a UPDATE con FROM:

```sql
DELETE FROM logs l
USING logs_archivo la
WHERE l.id = la.log_id
  AND la.fecha < '2024-01-01';
```

## RETURNING

RETURNING es una de las caracteristicas mas practicas de PostgreSQL. Permite que INSERT, UPDATE y DELETE devuelvan datos de las filas afectadas como si fueran un SELECT, eliminando la necesidad de consultas posteriores para obtener valores generados (SERIAL, UUID, defaults):

```sql
INSERT INTO pedidos (cliente_id, total)
VALUES (1, 299.99)
RETURNING id, created_at;

UPDATE productos
SET stock = stock - 1
WHERE id = 10
RETURNING nombre, stock AS nuevo_stock;

DELETE FROM tareas
WHERE completada = true
RETURNING id, titulo, completada_en;
```

RETURNING puede devolver multiples filas. En combinacion con ON CONFLICT, permite saber que ocurrio en cada fila de un UPSERT masivo:

```sql
INSERT INTO eventos (session_id, tipo, payload)
VALUES ('abc', 'click', '{"x": 10}'), ('def', 'view', '{"page": "/"}')
ON CONFLICT (session_id, tipo) DO UPDATE SET
    payload = EXCLUDED.payload,
    updated_at = now()
RETURNING *, (xmax = 0) AS fue_insertado;
```

El truco xmax = 0 merece explicacion: xmax es una columna del sistema MVCC que vale 0 para filas recien insertadas (todavia no eliminadas/actualizadas), permitiendo distinguir filas nuevas de actualizadas en un RETURNING de UPSERT.

