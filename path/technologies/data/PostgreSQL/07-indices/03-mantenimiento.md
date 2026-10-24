# Mantenimiento de indices

Los indices no son estructuras estaticas. Con el tiempo, las inserciones, actualizaciones y eliminaciones fragmentan el arbol B-Tree, inflan el indice con entradas muertas y degradan el rendimiento de las consultas. Mantener los indices en estado optimo requiere reconstrucciones periodicas, monitorizacion de su uso y eliminacion de aquellos que ya no aportan beneficio.

## Reconstruccion de indices con REINDEX

El comando `REINDEX` reconstruye un indice desde cero, eliminando la fragmentacion y recuperando el espacio ocupado por entradas obsoletas. Es la operacion de mantenimiento mas directa para restaurar el rendimiento de un indice degradado.

```sql
REINDEX INDEX idx_usuarios_email;
```

`REINDEX` bloquea el indice para escrituras durante su ejecucion (adquiere un lock `ACCESS EXCLUSIVE` sobre la tabla). Para entornos productivos donde el downtime no es aceptable, PostgreSQL ofrece `REINDEX CONCURRENTLY` desde la version 12:

```sql
REINDEX INDEX CONCURRENTLY idx_usuarios_email;
```

Esta variante crea un nuevo indice en paralelo mientras el antiguo sigue disponible para lecturas y escrituras, luego intercambia ambos atomicamente y elimina el antiguo. El proceso consume mas recursos y tarda mas, pero evita bloquear la tabla.

`REINDEX` admite varios niveles de granularidad:

```sql
REINDEX TABLE pedidos;                    -- Todos los indices de una tabla
REINDEX SCHEMA public;                     -- Todos los indices de un esquema
REINDEX DATABASE mi_base;                 -- Todos los indices de la base de datos
REINDEX INDEX CONCURRENTLY idx_problema;  -- Sin bloqueo (desde PG 12)
```

> `REINDEX TABLE` adquiere un lock exclusivo sobre la tabla. Si no se tolera downtime, `REINDEX INDEX CONCURRENTLY` sobre cada indice por separado es la opcion correcta.

### Cuando reconstruir

La reconstruccion esta justizada cuando:

- El indice ha crecido desproporcionadamente respecto a los datos reales. Un B-Tree puede inflarse si hay muchas actualizaciones que generan `dead tuples` en el indice.
- El rendimiento de las consultas que usan el indice se ha degradado, detectable mediante `EXPLAIN ANALYZE` y comparando el coste estimado con el real.
- Se ha producido una corrupcion del indice (errores en los logs indicando `broken index`).
- Despues de una operacion masiva de carga, borrado o actualizacion de datos.

## Estadisticas de uso con pg_stat_user_indexes

PostgreSQL expone estadisticas detalladas sobre cada indice en la vista del sistema `pg_stat_user_indexes`. Esta vista permite identificar que indices se usan, cuales no y cual es el costo de mantenimiento de cada uno.

```sql
SELECT
    schemaname,
    tablename,
    indexrelname AS indice,
    idx_scan AS lecturas,
    idx_tup_read AS tuplas_leidas,
    idx_tup_fetch AS tuplas_recuperadas
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;
```

Las columnas mas relevantes:

- `idx_scan`: numero de `Index Scan` realizados sobre el indice. Un valor cercano a cero indica que el indice apenas se usa.
- `idx_tup_read`: tuplas leidas del indice.
- `idx_tup_fetch`: tuplas recuperadas de la tabla a traves del indice.

La combinacion de `idx_scan` y `idx_tup_fetch` —junto con el tamaño del indice obtenido de `pg_indexes_size()`— da una medida clara del retorno de la inversion en espacio y escritura.

```sql
SELECT
    indexrelname AS indice,
    pg_size_pretty(pg_indexes_size(relid)) AS tamano,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY pg_indexes_size(relid) DESC;
```

> Las estadisticas de `pg_stat_user_indexes` se acumulan desde el ultimo reinicio o desde que se ejecuto `pg_stat_reset()`. Para un analisis preciso, conviene resetear las estadisticas tras una operacion de mantenimiento y observar el periodo de actividad normal.

### Vista complementaria: pg_stat_all_indexes

`pg_stat_all_indexes` incluye tanto indices de usuario como del catalogo del sistema. Es util para diagnosticar problemas a nivel de base de datos, pero en la mayoria de los casos `pg_stat_user_indexes` es suficiente.

## Identificar y eliminar indices no utilizados

Un indice que no se usa es puro lastre: ocupa espacio en disco, ralentiza las escrituras y consume recursos en el buffer cache sin aportar beneficio. La estrategia para eliminarlos combina las estadisticas de uso con el conocimiento del schema.

```sql
-- Indices sin escaneos desde el ultimo reseteo de estadisticas
SELECT
    schemaname,
    tablename,
    indexrelname AS indice,
    pg_size_pretty(pg_indexes_size(relid)) AS tamano
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_indexes_size(relid) DESC;
```

Un indice con `idx_scan = 0` no significa necesariamente que sea inutil: puede que cubra un patron de consulta estacional o que la aplicacion no haya ejecutado ciertas queries desde el ultimo reseteo. Por eso, antes de eliminar un indice conviene:

1. Verificar que el periodo de observacion cubre un ciclo completo de uso de la aplicacion (al menos una semana, preferiblemente un mes).
2. Confirmar que el indice no es la implementacion fisica de una `UNIQUE CONSTRAINT` o `PRIMARY KEY` (esos indices son necesarios para la integridad referencial aunque no aparezcan en `idx_scan`).
3. Comprobar con `EXPLAIN` si alguna consulta importante depende de ese indice.

```sql
-- Verificar si un indice respalda una restriccion
SELECT
    i.indexrelid::regclass AS indice,
    c.conname AS constraint_name,
    c.contype AS tipo_restriccion
FROM pg_index i
JOIN pg_constraint c ON i.indexrelid = c.conindid
WHERE i.indexrelid = 'idx_usuarios_email'::regclass;
```

Para eliminar un indice identificado como no necesario:

```sql
DROP INDEX IF EXISTS idx_inutil;
```

Si hay dudas sobre el impacto de la eliminacion, se puede marcar el indice como no usable temporalmente con `ALTER INDEX ... UNUSABLE` (solo en algunas versiones) o, mas practico, mantener el indice pero documentar la sospecha de inutilidad y monitorizar durante otro ciclo.

### Indices duplicados y redundantes

Los indices duplicados son aquellos que indexan exactamente las mismas columnas en el mismo orden. Los redundantes son indices que son un subconjunto de otro indice multicolumna. PostgreSQL no previene su creacion, pero detectarlos es sencillo:

```sql
SELECT
    a.indexrelid::regclass AS indice_a,
    b.indexrelid::regclass AS indice_b,
    a.indkey,
    b.indkey
FROM pg_index a
JOIN pg_index b ON a.indrelid = b.indrelid
    AND a.indexrelid < b.indexrelid
    AND a.indkey LIKE b.indkey || '%';
```

Mantener la disciplina de nombrar los indices con un patron que refleje su proposito (por ejemplo, `idx_tabla_columna`) ayuda a identificar rapidamente duplicados y a decidir que indices conservar.

