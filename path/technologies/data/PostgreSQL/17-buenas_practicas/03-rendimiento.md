# Rendimiento

El rendimiento en PostgreSQL no se obtiene de una sola configuracion ni de un conjunto de reglas estaticas: es el resultado de la observacion continua del planificador de consultas, la indexacion adecuada y las practicas de acceso a datos que minimizan el trabajo del motor. Comprender como PostgreSQL ejecuta las consultas es el primer paso para optimizarlas.

## EXPLAIN ANALYZE

La herramienta fundamental para diagnosticar el rendimiento de una consulta es `EXPLAIN ANALYZE`. Muestra el plan de ejecucion que el optimizador ha elegido y, para cada nodo, el tiempo real, el numero de filas y el numero de iteraciones.

```sql
EXPLAIN (ANALYZE, BUFFERS, TIMING) 
SELECT c.nombre, COUNT(p.id) AS total_pedidos
FROM clientes c
LEFT JOIN pedidos p ON p.cliente_id = c.id
WHERE c.esta_activo = true
GROUP BY c.id;
```

Los parametros clave que debe mostrar el plan:

- **`Seq Scan`**: lectura secuencial de toda la tabla. Aceptable solo para tablas pequenas (< 10 000 filas) o cuando se accede a mas del 10-15% de las filas.
- **`Index Scan`**: acceso por indice, tipicamente rapido para pocas filas. Cada fila requiere un acceso aleatorio al heap (la tabla).
- **`Index Only Scan`**: ideal: todas las columnas necesarias estan en el indice, evitando acceder al heap.
- **`Nested Loop`**: para cada fila de la tabla externa, busca en la interna. Eficiente con pocas filas.
- **`Hash Join`**: construye una tabla hash en memoria con una de las tablas. Rapido para joins entre conjuntos medianos.
- **`Merge Join`**: ordena ambas tablas por la clave de join y las recorre en paralelo. Util cuando los datos ya estan ordenados.

```sql
-- Identificar consultas lentas en una sesion
SELECT pid, state, query, wait_event_type, wait_event
FROM pg_stat_activity
WHERE state = 'active' AND query NOT LIKE '%pg_stat_activity%';
```

> Ejecutar `EXPLAIN ANALYZE` realmente ejecuta la consulta. Para sentencias `INSERT`/`UPDATE`/`DELETE`, usar `EXPLAIN (ANALYZE, BUFFERS)` dentro de una transaccion que se deshace con `ROLLBACK` para evitar efectos colaterales.

## Indices en claves foraneas

PostgreSQL no crea automaticamente indices en las columnas que forman una foreign key. Sin ese indice, cada operacion de insercion, actualizacion o eliminacion en la tabla referenciada dispara una busqueda secuencial en la tabla hija para verificar la integridad referencial.

```sql
CREATE TABLE pedidos (
    id          INTEGER PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    cliente_id  INTEGER NOT NULL REFERENCES clientes(id)
);
CREATE INDEX idx_pedidos_cliente_id ON pedidos (cliente_id);
```

La ausencia de este indice es una de las causas mas frecuentes de degradacion de rendimiento en bases de datos con multiples tablas relacionadas. La regla es simple: toda foreign key debe tener un indice en la columna local, a menos que exista un indice compuesto cuyo prefijo coincida con esa columna.

## Indices en WHERE, JOIN y ORDER BY

No todas las columnas merecen un indice. La decision debe basarse en las consultas reales que ejecuta la aplicacion, no en una especulacion previa.

```sql
-- Consulta frecuente:
SELECT id, nombre, email
FROM clientes
WHERE email = 'usuario@ejemplo.com';
-- Indice necesario:
CREATE UNIQUE INDEX idx_clientes_email ON clientes (email);

-- Ordenacion frecuente:
SELECT * FROM pedidos
WHERE cliente_id = 42
ORDER BY fecha DESC;
-- Indice compuesto optimo (filtra y ordena simultaneamente):
CREATE INDEX idx_pedidos_cliente_fecha ON pedidos (cliente_id, fecha DESC);
```

El orden de las columnas en un indice compuesto importa: las columnas de igualdad (`=`) deben ir primero, seguidas de las de rango (`>`, `<`, `BETWEEN`) y finalmente las de ordenacion (`ORDER BY`).

```sql
-- Indice optimo para: WHERE a = 1 AND b > 10 ORDER BY c
CREATE INDEX idx_tabla_a_b_c ON tabla (a, b, c);
```

> Un indice B-Tree puede devolver filas ya ordenadas, evitando un paso de ordenacion explicito (y su correspondiente uso de memoria y CPU). Si el `ORDER BY` coincide con el orden del indice, el plan mostrara `Index Scan Backward` en lugar de `Sort`.

## Evitar SELECT *

`SELECT *` es comodo en consultas exploratorias, pero perjudicial en produccion por varias razones:

1. Fuerza al motor a leer todas las columnas del heap aunque solo se necesiten dos, impidiendo el `Index Only Scan`.
2. Transmite mas datos entre la base de datos y la aplicacion, aumentando la latencia de red.
3. Acopla la aplicacion al esquema: anadir una columna a la tabla cambia el resultado de la consulta sin que el codigo lo espere.

```sql
-- Permite Index Only Scan si existe indice en (id, nombre, email)
SELECT id, nombre, email FROM clientes WHERE email = 'x@y.com';
-- Fuerza lectura del heap aunque email este indexado
SELECT * FROM clientes WHERE email = 'x@y.com';
```

## Paginacion con LIMIT y OFFSET

La paginacion con `LIMIT` y `OFFSET` es sencilla de implementar, pero se vuelve ineficiente cuando el `OFFSET` es grande: PostgreSQL debe generar y descartar todas las filas anteriores al offset.

```sql
-- Ineficiente en paginas avanzadas
SELECT * FROM pedidos ORDER BY id LIMIT 20 OFFSET 10000;
```

Alternativa: paginacion basada en cursor (keyset pagination), que usa una condicion `WHERE` sobre la columna por la que se ordena.

```sql
-- Eficiente independientemente de la profundidad
SELECT * FROM pedidos
WHERE id > 10020   -- ultimo id de la pagina anterior
ORDER BY id
LIMIT 20;
```

Esta tecnicamente se conoce como "paginacion por conjunto de claves" y es especialmente eficaz cuando la columna de ordenacion tiene un indice unico. Para ordenaciones por multiples columnas, la condicion debe incluir todas ellas.

## Prepared statements

Los prepared statements evitan que el planificador recompile el plan de ejecucion cada vez que se ejecuta una consulta parametrica. PostgreSQL admite dos niveles: las sentencias preparadas del lado del servidor (`PREPARE`/`EXECUTE`) y la preparacion implicita desde los drivers cliente.

```sql
PREPARE buscar_pedidos_cliente (INTEGER, DATE) AS
    SELECT id, total, fecha
    FROM pedidos
    WHERE cliente_id = $1 AND fecha >= $2
    ORDER BY fecha DESC;

EXECUTE buscar_pedidos_cliente(42, '2024-01-01');
```

Desde el cliente (ejemplo con psycopg2 en Python o libpq en C), los drivers tipicamente preparan la sentencia automaticamente tras varias ejecuciones, o el programador puede forzar la preparacion explicita.

El beneficio es mayor en consultas repetitivas con distintos parametros, como las generadas por un ORM. La preparacion reduce el overhead de parsing, analisis sintactico y optimizacion.

## Connection pooling

Establecer una conexion TCP a PostgreSQL es una operacion costosa: implica autenticacion, negociacion SSL, asignacion de memoria por parte del proceso backend (fork por conexion en el modelo tradiccional de PostgreSQL). El connection pooling mantiene un conjunto de conexiones reutilizables.

A nivel de aplicacion, herramientas como PgBouncer o Pgpool-II actuan como proxy entre los clientes y la base de datos, manteniendo abiertas un numero fijo de conexiones al servidor y asignandolas a los clientes bajo demanda.

```ini
# PgBouncer - configuracion tipica (pgbouncer.ini)
[databases]
mibd = host=127.0.0.1 port=5432 dbname=mibd

[pgbouncer]
listen_addr = 127.0.0.1
listen_port = 6432
pool_mode = transaction        ; cada transaccion obtiene una conexion
max_client_conn = 100
default_pool_size = 20         ; conexiones al servidor
```

El modo `transaction` es el mas equilibrado: cada conexion se devuelve al pool al finalizar la transaccion, permitiendo que multiples clientes compartan pocas conexiones reales al servidor.

> El numero maximo de conexiones que PostgreSQL puede manejar eficientemente esta limitado por `max_connections` (tipicamente 100-300). Mas alla de ese limite, el contexto switching y la contienda por recursos compartidos (buffer pool, locks) degradan el rendimiento. El pool permite que cientos de clientes compartan decenas de conexiones reales.

