# Monitoreo y administracion

Una base de datos productiva es un sistema vivo cuyo comportamiento cambia con la carga de trabajo, el volumen de datos y las condiciones del hardware. Monitorear no es una actividad reactiva que se activa cuando algo falla: es un proceso continuo que permite anticipar cuellos de botella, detectar regresiones de rendimiento y planificar la capacidad. PostgreSQL expone su estado interno a traves de un conjunto de vistas del catalogo `pg_stat_*` que ofrecen metrica granular sobre cada aspecto del servidor.

## Vistas de monitoreo

### pg_stat_activity

La vista mas consultada en el dia a dia del DBA. Muestra una fila por cada conexion activa (backend) con informacion sobre la consulta en ejecucion, su estado, la hora de inicio y el bloqueo que pueda estar esperando:

```sql
SELECT pid, usename, application_name, client_addr, state, query, wait_event_type, wait_event
FROM pg_stat_activity
WHERE state != 'idle';
```

Columnas clave:

- `pid`: identificador del proceso backend.
- `state`: estado actual (`active`, `idle`, `idle in transaction`, `fastpath function call`).
- `query`: texto de la consulta (truncado a 1024 bytes por defecto; configurable con `track_activity_query_size`).
- `wait_event_type` y `wait_event`: indican si el proceso esta esperando por un recurso (lock, IO, red, etc.). Son el punto de partida para diagnosticar contención.
- `backend_start` y `query_start`: marcas temporales de inicio.

> Una sesion en estado `idle in transaction` mantiene abierta una transaccion sin ejecutar consultas. Si ademas mantiene locks, puede bloquear a otros procesos. Es un patron problematico que suele indicar un error en la logica de la aplicacion.

### pg_stat_database

Estadisticas a nivel de base de datos: conexiones, transacciones confirmadas y revertidas, tuplas leidas y escritas, tiempo de ejecucion y numero de conflictos de recuperacion:

```sql
SELECT datname, numbackends, xact_commit, xact_rollback,
       tup_fetched, tup_inserted, tup_updated, tup_deleted,
       blks_read, blks_hit, hit_ratio
FROM pg_stat_database;
```

La columna `blks_hit` cuenta accesos a bloques encontrados en la cache compartida, y `blks_read` los que requirieron IO a disco. La relacion `blks_hit / (blks_hit + blks_read)` da el _cache hit ratio_, un indicador fundamental de la eficiencia de la memoria compartida.

### pg_stat_user_tables y pg_stat_user_indexes

Estadisticas detalladas por tabla e indice:

```sql
-- Tablas mas escaneadas secuencialmente (candidatas a indexacion)
SELECT relname, seq_scan, seq_tup_read, idx_scan
FROM pg_stat_user_tables
WHERE seq_scan > 0
ORDER BY seq_tup_read DESC;

-- Indices no utilizados (candidatos a eliminacion)
SELECT indexrelname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
WHERE idx_scan = 0;
```

Estas vistas permiten decisiones basadas en datos: que tablas necesitan indices, cuales tienen indices infrautilizados que solo ralentizan las escrituras, y cuales acumulan bloques muertos por actualizaciones masivas.

### pg_stat_bgwriter

Estadisticas del proceso background writer, que sincroniza buffers sucios con el disco:

```sql
SELECT checkpoints_timed, checkpoints_req, checkpoint_write_time,
       buffers_checkpoint, buffers_clean, buffers_backend,
       buffers_alloc
FROM pg_stat_bgwriter;
```

Una proporcion alta de `checkpoints_req` frente a `checkpoints_timed` indica que los checkpoint ocurren porque se llena el archivo WAL y no por tiempo, lo que sugiere ajustar `max_wal_size` o `checkpoint_completion_target`.

### pg_stat_wal

 desde PostgreSQL 14, estadisticas del Write-Ahead Log:

```sql
SELECT wal_records, wal_fpi, wal_bytes, wal_write, wal_sync
FROM pg_stat_wal;
```

`wal_fpi` (full page images) es relevante en replicacion: un valor alto indica muchas paginas completas escritas en el WAL, a menudo por la configuracion `full_page_writes` que protege contra paginas parcialmente escritas tras un fallo.

## Administracion

### Terminacion de conexiones

Cuando una consulta se eterniza o una sesion bloquea recursos, es necesario intervenir:

```sql
-- Cancelar la consulta actual sin cerrar la conexion
SELECT pg_cancel_backend(pid);

-- Terminar la conexion completamente
SELECT pg_terminate_backend(pid);
```

`pg_cancel_backend` envia una senal SIGINT al backend, que intenta abortar la consulta en curso de forma cooperativa. `pg_terminate_backend` envia SIGTERM y cierra la conexion, lo que fuerza un rollback de la transaccion activa. Es buena practica intentar primero `pg_cancel_backend` y solo recurrir a `pg_terminate_backend` si el primero no funciona.

```sql
-- Terminar todas las conexiones de una base de datos excepto la propia
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = 'bd_objetivo'
  AND pid <> pg_backend_pid();
```

### Tamanos de base de datos y objetos

```sql
-- Tamano de la base de datos actual
SELECT pg_database_size(current_database());

-- Tamano de una tabla (datos + indices + toast)
SELECT pg_total_relation_size('nombre_tabla');

-- Desglose: tabla, indices, toast
SELECT pg_table_size('nombre_tabla'),
       pg_indexes_size('nombre_tabla'),
       pg_total_relation_size('nombre_tabla');
```

Estas funciones devuelven bytes. Pueden combinarse con `pg_size_pretty()` para formato legible:

```sql
SELECT pg_size_pretty(pg_database_size(current_database()));
```

### VACUUM FULL y REINDEX

Con el tiempo, las tablas sufren fragmentacion (_bloat_) debido a las actualizaciones y eliminaciones. PostgreSQL usa MVCC, lo que significa que las filas muertas ocupan espacio hasta que VACUUM las reclama.

`VACUUM` estandar (sin FULL) marca el espacio como reusable pero no lo devuelve al sistema operativo. `VACUUM FULL` compacta la tabla y libera espacio al sistema, pero bloquea la tabla con un `ACCESS EXCLUSIVE LOCK` durante toda la operacion:

```sql
VACUUM FULL tabla_problematica;
```

`REINDEX` reconstruye un indice desde cero, eliminando la fragmentacion:

```sql
REINDEX TABLE tabla_problematica;
REINDEX INDEX indice_fragmentado;

-- Reconstruir todos los indices de todas las tablas (concurrentemente desde PG12)
REINDEX SCHEMA public CONCURRENTLY;
```

Desde PostgreSQL 12, `REINDEX CONCURRENTLY` evita bloqueos exclusivos prolongados al crear el nuevo indice en paralelo y reemplazar el antiguo cuando termina.

### COMMENT ON

Anadir documentacion inline a los objetos de la base de datos:

```sql
COMMENT ON TABLE clientes IS 'Datos maestros de clientes del sistema CRM';
COMMENT ON COLUMN clientes.email IS 'Correo corporativo, unico por cliente';
COMMENT ON DATABASE produccion IS 'Base de datos de produccion';
```

Los comentarios se almacenan en el catalogo y son visibles mediante `\d+` en psql o consultando `pg_description`:

```sql
SELECT objoid, objsubid, description FROM pg_description;
```

## Paginas de estado

### pg_stat_replication

Muestra el estado de los servidores en replicacion fisica:

```sql
SELECT application_name, client_addr, state, sync_state,
       replay_lag, write_lag, flush_lag, replay_lag
FROM pg_stat_replication;
```

Las columnas `*_lag` (desde PG10) indican el retraso en segundos entre el primario y el replico. `sync_state` puede ser `sync`, `async` o `potential`.

### pg_locks

Informacion sobre todos los locks activos en el sistema. Es fundamental para diagnosticar bloqueos y deadlocks:

```sql
SELECT pid, locktype, relation::REGCLASS, mode, granted, waitstart
FROM pg_locks
WHERE NOT granted
ORDER BY waitstart;
```

Las filas con `granted = false` son procesos que estan esperando un lock. Combinando `pg_locks` con `pg_stat_activity` se obtiene la cadena completa de bloqueo:

```sql
WITH bloqueos AS (
    SELECT pid, relation::REGCLASS AS objeto,
           mode, granted, waitstart
    FROM pg_locks
    WHERE locktype = 'relation'
)
SELECT b1.pid AS bloqueado, b2.pid AS bloqueante, b1.objeto
FROM bloqueos b1
JOIN bloqueos b2 ON b1.objeto = b2.objeto
                 AND b1.granted = false
                 AND b2.granted = true;
```

### pg_stat_progress_*

Vistas que exponen el progreso de operaciones de larga duracion. Cada operacion tiene su propia vista:

- `pg_stat_progress_vacuum`: progreso de VACUUM.
- `pg_stat_progress_create_index`: construccion de indices.
- `pg_stat_progress_cluster`: ejecucion de CLUSTER.
- `pg_stat_progress_copy`: carga con COPY.

```sql
SELECT pid, phase, heap_blks_total, heap_blks_scanned,
       heap_blks_vacuumed, index_vacuum_count
FROM pg_stat_progress_vacuum;
```

## Grafana + Prometheus

### postgres_exporter

Para un monitoreo continuo y centralizado, la combinacion Prometheus + Grafana es el estandar de facto en el ecosistema open source. `postgres_exporter` es un recolector de metricas que expone las vistas `pg_stat_*` como metricas Prometheus:

```bash
# Ejecucion basica
DATA_SOURCE_NAME="postgresql://usuario:password@localhost:5432/bd?sslmode=disable"
./postgres_exporter --web.listen-address=:9187
```

Configuracion tipica en `postgresql.yml` para metricas personalizadas:

```yaml
pg_stat_statements:
  query: "SELECT queryid, query, calls, total_exec_time FROM pg_stat_statements"
  metrics:
    - calls:
        usage: "COUNTER"
        description: "Numero de ejecuciones"
    - total_exec_time:
        usage: "COUNTER"
        description: "Tiempo total de ejecucion (ms)"
```

### Dashboards

El repositorio de `postgres_exporter` incluye dashboards importables en Grafana que cubren:

- Conexiones activas por estado y base de datos.
- Cache hit ratio y tuplas le/escritas por segundo.
- Transacciones commit/rollback, checkpoint, WAL generado.
- Replicacion: lag, estado de los replicos, archivos WAL pendientes.
- Locks: contencion y esperas.
- Tamanos de bases de datos y crecimiento historico.

### Alertas tipicas

Con Prometheus y Alertmanager se definen alertas sobre las metricas expuestas:

```yaml
groups:
  - name: postgresql
    rules:
      - alert: PostgreSQLDown
        expr: pg_up == 0
        for: 1m
        labels: { severity: critical }

      - alert: PostgreSQLReplicaLag
        expr: pg_replication_lag_seconds > 300
        for: 5m
        labels: { severity: warning }

      - alert: PostgreSQLHighConnections
        expr: sum(pg_stat_database_numbackends) > 200
        for: 2m
        labels: { severity: warning }

      - alert: PostgreSQLCacheHitRatioLow
        expr: (sum(rate(pg_stat_database_blks_hit[5m])) / (sum(rate(pg_stat_database_blks_hit[5m])) + sum(rate(pg_stat_database_blks_read[5m])))) < 0.95
        for: 10m
        labels: { severity: critical }
```

### Monitoreo de replicacion

La replicacion requiere vigilancia especifica ademas de las metricas generales. Los indicadores clave son:

- **Lag de replicacion:** `pg_stat_replication.replay_lag` combinado con `pg_wal_lsn_diff()`.
- **Estado del replico:** si `pg_stat_replication.state` es `streaming`, el replico esta al dia.
- **Archivos WAL acumulados:** si el replico se retrasa, el primario acumula WAL sin liberar, lo que puede llenar el disco.
- **Restore count:** en `pg_stat_bgwriter`, un numero alto de restauraciones desde archivo WAL indica que los replicos estan recuperando desde archivos archivados en lugar de aplicar cambios en tiempo real, senal de que el lag es excesivo.

