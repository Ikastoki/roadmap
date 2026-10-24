# VACUUM en PostgreSQL

PostgreSQL implementa MVCC (Multi-Version Concurrency Control): cuando una fila se actualiza o elimina, la version anterior no se borra fisicamente de la pagina de disco, sino que se marca como muerta (`dead tuple`). Este mecanismo permite que lecturas concurrentes vean la foto consistente de la base de datos sin bloqueos, pero genera la necesidad de limpiar esas versiones obsoletas. VACUUM es el proceso que recupera ese espacio y actualiza el mapa de visibilidad.

## VACUUM estandar

`VACUUM` (sin mas) marca el espacio ocupado por tuples muertos como disponible para reutilizacion dentro de la misma pagina. No reduce el tamaño del archivo en disco ni compacta las paginas.

```sql
VACUUM pedidos;
```

El vaciado estandar permite que nuevas inserciones y actualizaciones reutilicen el espacio liberado, evitando que la tabla crezca indefinidamente. Es una operacion que puede ejecutarse concurrentemente con lecturas y escrituras (no bloquea).

Para todas las bases de datos:

```sql
VACUUM;
```

## VACUUM FULL

`VACUUM FULL` es una operacion mas agresiva: reescribe toda la tabla en un nuevo archivo, compactando las paginas y devolviendo el espacio liberado al sistema operativo.

```sql
VACUUM FULL pedidos;
```

A diferencia de `VACUUM`, `VACUUM FULL` **bloquea** la tabla (adquiere un lock exclusivo) durante toda la operacion. Es una operacion costosa en E/S y tiempo que debe planificarse en ventanas de mantenimiento.

> `VACUUM FULL` no suele ser necesario si `autovacuum` funciona correctamente. Solo se justifica cuando una tabla ha crecido mucho y luego se ha eliminado un porcentaje grande de sus filas, o despues de operaciones masivas de DELETE/UPDATE sin VACUUM intermedio.

## VACUUM ANALYZE

Combina el vaciado con la actualizacion de estadisticas para el optimizador:

```sql
VACUUM ANALYZE pedidos;
```

Es equivalente a ejecutar `VACUUM pedidos` seguido de `ANALYZE pedidos`, pero con un solo pase sobre los datos.

## Dead tuples y pg_stat_user_tables

La vista `pg_stat_user_tables` expone varias columnas relacionadas:

- `n_live_tup`: numero estimado de filas vivas.
- `n_dead_tup`: numero estimado de filas muertas.
- `n_tup_ins`, `n_tup_upd`, `n_tup_del`: contadores acumulados de operaciones.
- `last_autovacuum`, `last_vacuum`: marcas temporales del ultimo VACUUM.
- `vacuum_count`, `autovacuum_count`: contadores de ejecuciones.

Una tabla con `n_dead_tup` persistentemente alto (miles o millones) indica que VACUUM no esta al dia. La consulta tipica para detectar tablas problemáticas:

```sql
SELECT relname, n_dead_tup, n_live_tup,
       round(100.0 * n_dead_tup / (n_live_tup + n_dead_tup + 1), 2) AS dead_pct
FROM pg_stat_user_tables
WHERE n_live_tup > 0
ORDER BY dead_pct DESC
LIMIT 10;
```

## autovacuum

PostgreSQL incluye un demonio interno llamado `autovacuum` que ejecuta VACUUM y ANALYZE automaticamente segun la actividad de cada tabla. Se configura en `postgresql.conf` con estos parametros principales:

- `autovacuum = on`: activa el proceso.
- `autovacuum_naptime`: intervalo entre comprobaciones (por defecto 1 minuto).
- `autovacuum_vacuum_threshold`: numero minimo de tuples muertos para disparar VACUUM (por defecto 50).
- `autovacuum_vacuum_scale_factor`: fraccion de la tabla que, al acumularse como tuples muertos, dispara VACUUM (por defecto 0.2, es decir, 20 %).
- `autovacuum_analyze_threshold` y `autovacuum_analyze_scale_factor`: analogos para ANALYZE.

La condicion de disparo es: `n_dead_tup > threshold + scale_factor * n_live_tup`.

En tablas muy grandes (cientos de millones de filas), el factor de escala del 20 % puede significar decenas de millones de tuples muertos antes de que se active autovacuum. Para estos casos conviene reducir `autovacuum_vacuum_scale_factor` o establecer valores por tabla mediante opciones de almacenamiento:

```sql
ALTER TABLE pedidos SET (autovacuum_vacuum_scale_factor = 0.01);
```

O directamente con un threshold fijo:

```sql
ALTER TABLE pedidos SET (autovacuum_vacuum_threshold = 1000);
```

> Cuando autovacuum no da abasto, se acumulan dead tuples. Ademas de ocupar espacio, degradan el rendimiento de los Index Scan (que deben saltarse los tuples muertos) y pueden provocar "wraparound" de XID, una condicion que obliga a detener la base de datos para un VACUUM de emergencia.

