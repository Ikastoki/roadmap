# EXPLAIN y Estadisticas en PostgreSQL

El optimizador de PostgreSQL genera un plan de ejecucion para cada consulta antes de ejecutarla. Comprender como leer e interpretar ese plan es la habilidad mas importante para diagnosticar problemas de rendimiento. Sin esta capacidad, cualquier ajuste de configuracion o cambio estructural es esencialmente ciego.

## El plan de ejecucion con EXPLAIN

`EXPLAIN` muestra el plan que el optimizador ha construido sin llegar a ejecutar la consulta. La salida es un arbol de nodos, donde cada nodo representa una operacion (escaneo, join, ordenacion, etc.) y fluye de las hojas hacia la raiz.

```sql
EXPLAIN SELECT * FROM usuarios WHERE email LIKE '%gmail.com';
```

La salida tipica muestra para cada nodo:

- **cost**: dos valores separados por `..` -- el costo de arranque (startup) y el costo total. La unidad es arbitraria (paginas de disco leidas secuencialmente), pero es consistente para comparar planes.
- **rows**: estimacion del numero de filas que producira ese nodo.
- **width**: tamaño promedio en bytes de cada fila.

Los costos startup y total merecen atencion separada. El startup cost representa el trabajo que debe hacerse antes de poder producir la primera fila (por ejemplo, ordenar un conjunto antes de devolverlo). El total cost es el trabajo completo estimado. Una consulta con startup alto pero total bajo puede ser adecuada si solo se necesitan las primeras filas (con `LIMIT` o en un cursor).

## EXPLAIN ANALYZE

`EXPLAIN ANALYZE` ejecuta realmente la consulta (con efectos secundarios incluidos en escrituras) y muestra los tiempos reales junto con las estimaciones. Las diferencias entre el costo estimado y el real son la pista principal para detectar estadisticas desactualizadas o estimaciones incorrectas.

```sql
EXPLAIN ANALYZE SELECT * FROM pedidos WHERE fecha >= '2025-01-01';
```

La salida anade:

- **actual time**: tiempo real de arranque y tiempo total en milisegundos.
- **rows**: filas reales producidas.
- **loops**: cuantas veces se ejecuto ese nodo (relevante en subconsultas correlacionadas o nested loops).

Un nested loop que se ejecuta miles de veces es casi siempre un problema de diseno.

## BUFFERS

El modificador `BUFFERS` (usado junto con `ANALYZE`) muestra cuantas paginas fueron leidas o escritas desde el buffer pool (`shared_buffers`) y desde el disco:

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM logs WHERE fecha > '2025-06-01';
```

Los numeros clave son `shared hit` (pagina encontrada en cache) frente a `read` (lectura fisica de disco). Una proporcion alta de `read` indica que la consulta no esta siendo servida desde la cache y que probablemente hacen falta mas indices o un ajuste de `shared_buffers`.

## FORMAT JSON

`EXPLAIN (FORMAT JSON)` (o combinado con `ANALYZE, BUFFERS`) produce una salida estructurada que herramientas externas y funciones como `pg_show_plans` pueden consumir:

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) SELECT * FROM clientes WHERE id = 42;
```

## Tipos de escaneo: Seq Scan vs Index Scan

- **Seq Scan** (sequential scan): lee toda la tabla secuencialmente. Es eficiente cuando la tabla es pequena o cuando la consulta recupera un porcentaje alto de filas (tipicamente > 5-10 %). El costo crece linealmente con el tamaño de la tabla.

- **Index Scan**: recorre el indice para localizar las filas y luego accede a la tabla (si es necesario). Cuando el indice contiene todos los datos necesarios, PostgreSQL puede realizar un **Index Only Scan** que evita el acceso a la tabla.

- **Bitmap Scan**: combina varios indices. Primero construye un bitmap de paginas candidatas en memoria y luego accede a las filas. Es util cuando se usan condiciones en varias columnas que tienen indices independientes.

El optimizador decide entre estos basandose en las estadisticas y los parametros de costo como `random_page_cost` y `effective_cache_size`.

## Estrategias de join: Hash Join, Merge Join y Nested Loop

- **Nested Loop**: para cada fila del conjunto externo, busca las correspondientes en el interno. Es eficiente cuando el conjunto externo es pequeno y el interno tiene un indice. Su costo es O(n * m) en el peor caso. Se vuelve problematico cuando uno de los conjuntos es grande o cuando no hay indice y se fuerza un Seq Scan interno repetido.

- **Hash Join**: construye una tabla hash en memoria con el conjunto mas pequeno y luego escanea el mayor buscando coincidencias. Es la opcion preferida para joins de volumen medio-grande sin indices adecuados. El startup cost puede ser alto por la construccion de la hash, pero el total suele ser mejor que un Nested Loop.

- **Merge Join**: ordena ambos conjuntos por la clave de join y luego los recorre en paralelo. Requiere que los datos esten ordenados previamente (por un indice o por un Sort explicito). Compite con Hash Join cuando los datos ya estan ordenados o cuando la clave de join permite una ordenacion barata.

## Estadisticas del sistema

PostgreSQL expone multiples vistas del catalogo para monitorizar el rendimiento:

**pg_stat_user_tables**

Muestra, por tabla, contadores de accesos secuenciales (`seq_scan`, `seq_tup_read`), accesos por indice (`idx_scan`, `idx_tup_fetch`), y operaciones DML (`n_tup_ins`, `n_tup_upd`, `n_tup_del`). La relacion entre `seq_scan` y `idx_scan` indica si una tabla recibe escaneos secuenciales excesivos.

**pg_stat_user_indexes**

Contiene contadores de uso de cada indice (`idx_scan`). Un indice con `idx_scan` cercano a cero probablemente es redundante y consume recursos en escritura sin beneficio en lectura.

**pg_stat_activity**

Muestra las conexiones activas, la consulta en ejecucion, el estado (active, idle, idle in transaction, etc.) y el tiempo de ejecucion. Es la primera vista que se consulta cuando el sistema presenta lentitud o bloqueos.

**pg_stat_statements**

Requiere habilitar la extension (`CREATE EXTENSION pg_stat_statements`). Almacena estadisticas acumuladas de todas las consultas ejecutadas: tiempo total, tiempo medio, numero de ejecuciones, bloques leidos y escritos, y filas afectadas. Es la herramienta fundamental para identificar las consultas mas costosas del sistema.

```sql
SELECT query, calls, total_exec_time / calls AS avg_ms, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

## ANALYZE y las estadisticas del optimizador

`ANALYZE` recorre una muestra de las filas de una tabla y actualiza las estadisticas en `pg_statistic`. Estas estadisticas incluyen la distribucion de valores (histogramas), la correlacion fisica, la nulabilidad y la densidad de valores comunes. Sin estadisticas actualizadas, el optimizador trabaja con suposiciones por defecto que suelen producir planes pesimos.

```sql
ANALYZE clientes;
```

`VACUUM ANALYZE` combina la limpieza de tuples muertos con la actualizacion de estadisticas en un solo pase.

Las estadisticas obsoletas son la causa mas frecuente de degradacion repentina del rendimiento. Cuando una consulta que funcionaba bien empieza a usar Seq Scan o cambia a un plan drasticamente distinto, el primer paso es ejecutar `ANALYZE` en las tablas involucradas.

