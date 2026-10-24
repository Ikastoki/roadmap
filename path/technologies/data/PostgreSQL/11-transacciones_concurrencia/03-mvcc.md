# MVCC: control de concurrencia multiversion

PostgreSQL implementa MVCC (Multiversion Concurrency Control) como su mecanismo fundamental para gestionar el acceso concurrente a los datos. En lugar de bloquear lecturas cuando una escritura esta en curso, MVCC mantiene multiples versiones fisicas de cada fila (tuples) y le presenta a cada transaccion la version que le corresponde segun su snapshot. Este diseno, heredado de Postgres original, permite que los lectores nunca bloqueen a los escritores ni viceversa, una propiedad que pocos sistemas de bases de datos ofrecen de forma nativa. La contrapartida es que las versiones antiguas deben limpiarse periodicamente mediante el proceso VACUUM, y el sistema debe gestionar el espacio de identificadores de transaccion para evitar el agotamiento (wraparound).

## Versiones de datos por transaccion

Cada fila en PostgreSQL almacena en su cabecera dos campos fundamentales: `xmin` y `xmax`. `xmin` contiene el identificador de transaccion (xid) que creo esta version de la fila. `xmax` contiene el xid de la transaccion que la elimino o actualizo, o 0 si la version sigue vigente.

Cuando una sentencia `UPDATE` se ejecuta, PostgreSQL no modifica la fila original. En su lugar, marca el tuple existente con el xid de la transaccion actual en `xmax` (haciendolo invisible para futuras transacciones) e inserta un nuevo tuple con el mismo valor en `xmin` y con los datos modificados. Una `DELETE` simplemente asigna el xid actual a `xmax` sin crear un nuevo tuple.

```sql
-- Estado inicial: una fila con xmin = 100, xmax = 0
-- Actualizacion por transaccion con xid = 150
UPDATE cuentas SET saldo = 500 WHERE id = 1;

-- Ahora existen dos tuples fisicos:
-- Tuple 1: xmin = 100, xmax = 150 (invisible para transacciones con xid > 150)
-- Tuple 2: xmin = 150, xmax = 0   (visible para transacciones con xid >= 150)
```

Cada transaccion activa recibe un snapshot al comenzar (o al ejecutar la primera consulta, segun el nivel de aislamiento) que registra que transacciones estaban activas en ese momento. Este snapshot se almacena en la memoria compartida del servidor y se usa para determinar la visibilidad de cada tuple.

## Tuple visibility

La visibilidad de un tuple se determina comparando `xmin` y `xmax` con el snapshot de la transaccion y con el estado de las transacciones registrado en la CLOG (Commit Log). Las reglas basicas son:

- Un tuple es visible si `xmin` corresponde a una transaccion confirmada (o a la propia transaccion) y `xmax` es 0 o corresponde a una transaccion aun no confirmada.
- Un tuple es invisible si `xmin` corresponde a una transaccion aun no confirmada (no se ven cambios de otros hasta que confirmen) o si `xmax` corresponde a una transaccion confirmada que ya elimino la fila.

Este proceso se ejecuta para cada fila en cada consulta. PostgreSQL optimiza la evaluacion mediante Hint Bits: bits en la cabecera del tuple que cachean el resultado de la comprobacion tras la primera vez que se evalua. Un tuple con los hint bits correctamente establecidos evita tener que consultar la CLOG en accesos posteriores.

> Sin hint bits, cada vez que una consulta accede a un tuple debe ir a la CLOG para determinar si `xmin` y `xmax` estan confirmados o abortados. El VACUUM y la escritura desde los backends establecen estos hint bits, mejorando el rendimiento de lecturas futuras.

## VACUUM y dead tuples

Las versiones antiguas de filas (dead tuples) no se eliminan inmediatamente. Permancen en la tabla ocupando espacio y afectando al rendimiento de los indices. El proceso `VACUUM` (manual o automatico) las reclama.

```sql
-- VACUUM estandar: marca el espacio de dead tuples como reusable,
-- actualiza estadisticas, pero no compacta la tabla
VACUUM cuentas;

-- VACUUM FULL: reescribe la tabla completa, dejando solo los tuples vivos
-- Requiere un bloqueo ACCESS EXCLUSIVE y es una operacion pesada
VACUUM FULL cuentas;
```

`VACUUM` sin `FULL` no reduce el tamanio del archivo en disco; solo libera espacio dentro de las paginas existentes para que nuevas inserciones puedan reutilizarlo. `VACUUM FULL` reescribe la tabla en nuevas paginas, devolviendo espacio al sistema operativo, pero bloquea la tabla completamente durante la operacion.

El autovacuum launcher ejecuta VACUUM de forma automatica. Su comportamiento se controla con parametros como `autovacuum_vacuum_threshold` (numero minimo de tuples muertos antes de disparar) y `autovacuum_vacuum_scale_factor` (fraccion de la tabla). Por defecto, autovacuum se activa cuando hay mas de 50 tuples muertos mas el 20 % de la tabla.

```sql
-- Consultar cuando se vacio cada tabla por ultima vez y cuantos
-- tuples muertos tiene pendientes
SELECT relname, n_dead_tup, last_autovacuum, last_vacuum
FROM pg_stat_user_tables;
```

Un autovacuum insuficiente provoca *table bloat*: la tabla crece desmesuradamente, los indices se degradan, las consultas secuenciales tardan mas y el rendimiento general cae. En tablas muy activas puede ser necesario ajustar los parametros de autovacuum por tabla usando storage parameters:

```sql
ALTER TABLE cuentas SET (autovacuum_vacuum_scale_factor = 0.05,
                         autovacuum_vacuum_threshold = 1000);
```

## Transaction ID wraparound

Los xid en PostgreSQL son numeros de 32 bits (capacidad para unos 4000 millones de transacciones). Cuando se agota el espacio de xids, el sistema da la vuelta (wraparound) y los xids nuevos vuelven a empezar desde 3 (los valores 0, 1 y 2 estan reservados). Como las comparaciones de visibilidad usan aritmetica modular, un xid muy antiguo puede volverse a considerar "futuro" y hacer que tuples viejos parezcan no visibles, corrompiendo logicamente los datos.

Para evitarlo, PostgreSQL dispara un VACUUM forzoso (denominado *aggressive vacuum*) cuando el numero de xids disponibles baja de un umbral configurable. El parametro `vacuum_freeze_min_age` controla cuantos xids deben pasar antes de que un tuple sea marcado como *frozen*. Un tuple con `xmin` congelado se considera siempre visible para cualquier transaccion, sin importar el xid actual.

```sql
-- Ver el avance del xid actual y el porcentaje de wraparound
SELECT datname, age(datfrozenxid) AS edad_xid,
       round(100 * age(datfrozenxid) / 2^31::numeric, 2) AS porcentaje_wraparound
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
```

La columna `datfrozenxid` indica el xid minimo que aun no ha sido congelado en esa base de datos. Si la edad (diferencia entre el xid actual y `datfrozenxid`) se acerca a 2000 millones, PostgreSQL empezara a emitir advertencias en los logs:

```
WARNING:  database "produccion" must be vacuumed within 17700000 transactions
HINT:  To avoid a database shutdown, execute a full-database VACUUM.
```

Si no se actua, cuando la edad supera los 2000 millones, PostgreSQL entra en modo de prevencion de wraparound y detiene todas las transacciones que puedan generar nuevos xids, basicamente paralizando las escrituras en la base de datos. La unica salida es ejecutar un `VACUUM FREEZE` inmediato, que en bases de datos muy grandes puede tardar horas.

## pg_stat_user_tables

La vista `pg_stat_user_tables` proporciona metricas esenciales para monitorizar la salud de MVCC en cada tabla:

```sql
SELECT relname,
       n_live_tup,          -- tuples vivos estimados
       n_dead_tup,          -- tuples muertos estimados
       n_mod_since_analyze, -- modificaciones desde el ultimo ANALYZE
       last_vacuum,
       last_autovacuum,
       last_analyze,
       vacuum_count,
       autovacuum_count
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

Una tabla con `n_dead_tup` persistentemente alto y sin `last_autovacuum` reciente indica que el autovacuum no esta siguiendo el ritmo de las modificaciones. En ese caso conviene revisar la configuracion de autovacuum para esa tabla concreta o, durante una ventana de mantenimiento, ejecutar un `VACUUM FULL` si el bloat ya es severo.

> `n_dead_tup` es una estimacion del planificador, no un contador exacto. Para obtener el numero real de tuples muertos habria que ejecutar un VACUUM que los limpie y luego contar. No obstante, para monitorizacion la estimacion es suficiente: si marca consistentemente miles de dead tuples, hay un problema real.

## Gestion del espacio con VACUUM FULL y CLUSTER

Cuando el bloat es avanzado, `VACUUM FULL` reescribe la tabla y los indices desde cero, devolviendo el espacio al sistema operativo. Como alternativa, `CLUSTER` reordena la tabla segun un indice, tambien compactandola pero ademas dejando las filas ordenadas fisicamente, lo que beneficia las consultas por rango.

```sql
-- Compactar y reordenar por el indice idx_cuentas_fecha
CLUSTER cuentas USING idx_cuentas_fecha;
```

Ambas operaciones requieren un bloqueo `ACCESS EXCLUSIVE` y deben planificarse en ventanas de bajo uso, especialmente en tablas de varios cientos de gigabytes donde la reescritura puede prolongarse durante horas.

