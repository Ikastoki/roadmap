# Niveles de aislamiento y bloqueos

El aislamiento determina como una transaccion percibe los cambios realizados por otras transacciones concurrentes. PostgreSQL implementa los cuatro niveles definidos en el estandar SQL, aunque con diferencias respecto a lo que cabria esperar teoricamente. El control de concurrencia se completa con bloqueos explicitos (SELECT FOR UPDATE, LOCK TABLE) que permiten al desarrollador afinar el comportamiento cuando el mecanismo de MVCC no es suficiente, y con herramientas como pg_locks y la deteccion de deadlocks para diagnosticar y resolver conflictos entre sesiones.

## Niveles de aislamiento

### READ UNCOMMITTED

En el estandar SQL, READ UNCOMMITTED permite leer datos modificados pero aun no confirmados por otras transacciones (*dirty read*). PostgreSQL trata READ UNCOMMITTED como READ COMMITTED: el motor simplemente no implementa el nivel mas bajo del estandar porque su arquitectura MVCC hace que un *dirty read* sea imposible sin reescribir partes fundamentales del sistema. En la practica, READ UNCOMMITTED en PostgreSQL se comporta exactamente igual que READ COMMITTED.

### READ COMMITTED (por defecto)

READ COMMITTED es el nivel de aislamiento predeterminado. Cada sentencia dentro de la transaccion ve solo los datos que estaban confirmados en el momento en que la sentencia comienza su ejecucion. Si otra transaccion confirma cambios entre dos sentencias de la misma transaccion, la segunda sentencia vera esos nuevos datos.

```sql
-- Sesion A
BEGIN;
SELECT saldo FROM cuentas WHERE id = 1;  -- 100

-- Sesion B (concurrente)
BEGIN;
UPDATE cuentas SET saldo = 200 WHERE id = 1;
COMMIT;

-- Sesion A (misma transaccion)
SELECT saldo FROM cuentas WHERE id = 1;  -- 200 (ve el cambio de B)
COMMIT;
```

Este comportamiento puede producir anomalias de lectura: una transaccion que ejecuta la misma consulta dos veces puede obtener resultados distintos si entre medias otra transaccion confirma cambios (non-repeatable read) o si aparecen filas nuevas (phantom read). READ COMMITTED es el nivel adecuado para la mayoria de aplicaciones porque ofrece un equilibrio razonable entre consistencia y rendimiento.

> El comando `SET TRANSACTION ISOLATION LEVEL READ COMMITTED` no es necesario porque ya es el valor por defecto, pero puede incluirse para hacer explicito el comportamiento en codigo que deba ser portado a otras bases de datos.

### REPEATABLE READ

REPEATABLE READ presenta a la transaccion una instantanea (snapshot) de los datos tomada al inicio de la primera sentencia de la transaccion. Todas las consultas posteriores ven exactamente la misma foto, independientemente de los cambios que otras transacciones confirmen mientras tanto. Esto elimina los *non-repeatable read* y los *phantom read*.

```sql
-- Sesion A
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT saldo FROM cuentas WHERE id = 1;  -- 100

-- Sesion B
UPDATE cuentas SET saldo = 200 WHERE id = 1;
COMMIT;

-- Sesion A (misma transaccion)
SELECT saldo FROM cuentas WHERE id = 1;  -- 100 (sigue viendo la instantanea)
COMMIT;
```

A diferencia de otros motores, en REPEATABLE READ PostgreSQL no sufre *phantom reads*: el snapshot es fijo y las consultas con predicados devuelven siempre el mismo conjunto de filas. Sin embargo, si la transaccion intenta modificar una fila que otra transaccion ya modifico y confirmo, PostgreSQL lanza un error de serializacion:

```sql
ERROR:  could not serialize access due to concurrent update
```

La aplicacion debe capturar este error y reintentar la transaccion. Este diseno sacrifica la actualizacion ciega a cambio de garantizar que no se produzcan perdidas de actualizacion.

### SERIALIZABLE

SERIALIZABLE garantiza que el resultado de ejecutar varias transacciones concurrentes es equivalente a ejecutarlas una tras otra en algun orden serial. Internamente, PostgreSQL implementa Serializable Snapshot Isolation (SSI), un algoritmo que detecta conflictos entre las lecturas y escrituras de las transacciones mediante el seguimiento de *predicate locks* (bloqueos sobre condiciones de consulta, no sobre filas concretas).

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT sum(saldo) FROM cuentas WHERE sucursal = 'central';
-- Si otra transaccion inserta o modifica filas que afectan a ese predicado,
-- al hacer COMMIT se lanza:
-- ERROR:  could not serialize access due to read/write dependencies among transactions
COMMIT;
```

SERIALIZABLE impone el mayor costo de rendimiento y la mayor tasa de abortos, pero es el unico nivel que protege contra todas las anomalias posibles, incluyendo las de escritura fantasma y las derivadas de dependencias ciclicas entre transacciones.

## Dirty read, non-repeatable read y phantom read

| Anomalia | Descripcion | READ COMMITTED | REPEATABLE READ | SERIALIZABLE |
|---|---|---|---|---|
| Dirty read | Leer datos no confirmados | No | No | No |
| Non-repeatable read | Misma consulta, resultados distintos por cambios confirmados entre medias | Si | No | No |
| Phantom read | Aparicion de filas que cumplen un predicado tras ser insertadas por otra transaccion | Si | No (en PostgreSQL) | No |

PostgreSQL no padece dirty reads en ningun nivel gracias a MVCC. Los phantoms se eliminan en REPEATABLE READ porque el snapshot es estatico, a diferencia de otros motores donde REPEATABLE READ solo protege contra modificaciones de filas existentes pero no contra inserciones.

## Bloqueos explicitos: SELECT FOR UPDATE, FOR NO KEY UPDATE, FOR SHARE

MVCC permite que lectores y escritores no se bloqueen mutuamente, pero cuando una transaccion necesita asegurarse de que una fila no cambie antes de modificarla, debe bloquearla explicitamente. Las clausulas `FOR UPDATE`, `FOR NO KEY UPDATE`, `FOR SHARE` y `FOR KEY SHARE` se anaden al final de un `SELECT` y adquieren bloqueos sobre las filas devueltas.

`SELECT ... FOR UPDATE` bloquea las filas seleccionadas para que ningun otro bloqueo de escritura pueda operar sobre ellas. Cualquier otra transaccion que intente un `FOR UPDATE`, `FOR NO KEY UPDATE`, `FOR SHARE` o `DELETE` sobre esas filas quedara esperando hasta que la transaccion bloqueante termine.

```sql
BEGIN;
SELECT * FROM cuentas WHERE id = 1 FOR UPDATE;
-- Otra sesion no podra actualizar ni borrar la cuenta 1
UPDATE cuentas SET saldo = saldo - 50 WHERE id = 1;
COMMIT;  -- se libera el bloqueo
```

`FOR NO KEY UPDATE` es una version debil de `FOR UPDATE`: bloquea contra actualizaciones que modificarian claves, pero permite bloqueos `FOR KEY SHARE`. Se usa cuando se actualiza la fila pero no su clave primaria.

`FOR SHARE` bloquea la fila para que ninguna otra transaccion pueda actualizarla o borrarla, pero permite que otras transacciones tambien adquieran `FOR SHARE`. Diferencia clave respecto a `FOR UPDATE`: varias sesiones pueden tener `FOR SHARE` simultaneamente sobre la misma fila, pero ninguna puede modificarla.

`FOR KEY SHARE` es el mas debil: bloquea contra eliminaciones o cambios de clave, pero permite actualizaciones que no afecten a la clave. Es el bloqueo que PostgreSQL adquiere automaticamente en las claves foraneas al modificar la tabla referenciada.

> La eleccion del tipo de bloqueo importa para la concurrencia. Usar `FOR UPDATE` donde bastaria `FOR NO KEY UPDATE` o `FOR SHARE` puede reducir innecesariamente el paralelismo en aplicaciones con alta contención.

## LOCK TABLE

La sentencia `LOCK TABLE` bloquea una tabla completa, no filas individuales. Es una operacion pesada que solo se justifica en operaciones masivas como migraciones, reindexaciones o procesos batch que necesitan garantizar que ninguna otra sesion accede a la tabla mientras se transforman los datos.

```sql
BEGIN;
LOCK TABLE cuentas IN ACCESS EXCLUSIVE MODE;
-- Ninguna otra operacion (lectura o escritura) puede tocar la tabla
ALTER TABLE cuentas ADD COLUMN ultima_actualizacion timestamp;
COMMIT;
```

Los modos de bloqueo disponibles son, de menor a mayor restriccion: `ACCESS SHARE` (SELECT), `ROW SHARE` (SELECT FOR UPDATE), `ROW EXCLUSIVE` (INSERT, UPDATE, DELETE), `SHARE UPDATE EXCLUSIVE` (VACUUM sin FULL), `SHARE` (CREATE INDEX), `SHARE ROW EXCLUSIVE`, `EXCLUSIVE` y `ACCESS EXCLUSIVE`. Cada modo entra en conflicto con un conjunto concreto de otros modos; PostgreSQL resuelve automaticamente la compatibilidad y bloqueara la peticion si otro bloqueo incompatible esta activo.

## Deadlocks: deteccion y resolucion

Un deadlock ocurre cuando dos o mas transacciones esperan mutuamente por un recurso que la otra retiene, creando un ciclo de dependencia que nunca se resuelve por si solo. PostgreSQL incluye un detector de deadlocks que se ejecuta cada `deadlock_timeout` milisegundos (por defecto 1000 ms). Cuando detecta un ciclo, elige una de las transacciones implicadas como victimay aborta su transaccion con el error:

```sql
ERROR:  deadlock detected
DETAIL:  Process X waits for ShareLock on transaction Y; blocked by process Z.
Process Z waits for ShareLock on transaction Y; blocked by process X.
HINT:  See server log for query details.
```

La aplicacion debe capturar este error y reintentar la transaccion. La victima se selecciona basandose en un criterio interno: se aborta la transaccion que menos costo haya acumulado hasta el momento.

```sql
-- Sesion A
BEGIN;
UPDATE cuentas SET saldo = 100 WHERE id = 1;

-- Sesion B
BEGIN;
UPDATE cuentas SET saldo = 200 WHERE id = 2;

-- Sesion A (ahora espera por la fila que bloquea B)
UPDATE cuentas SET saldo = 300 WHERE id = 2;  -- se bloquea

-- Sesion B (intenta acceder a la fila que bloquea A, y se produce el deadlock)
UPDATE cuentas SET saldo = 400 WHERE id = 1;  -- deadlock detectado, B es abortada
```

Para prevenir deadlocks conviene acceder a los recursos siempre en el mismo orden (por ejemplo, actualizar las cuentas siempre por id ascendente) y mantener las transacciones lo mas cortas posible.

## pg_locks: diagnostico de bloqueos

La vista del sistema `pg_locks` expone en tiempo real todos los bloqueos activos en el servidor: tanto los que estan concedidos como los que estan en espera. Es la herramienta principal para diagnosticar problemas de contencion.

```sql
SELECT pid, locktype, relation::regclass, mode, granted,
       waitstart, virtualtransaction
FROM pg_locks
WHERE NOT granted;
```

Esta consulta muestra las sesiones que estan esperando un bloqueo sin conseguirlo. Combinada con `pg_stat_activity` se puede identificar la consulta concreta que causa la espera:

```sql
SELECT a.pid, a.query, a.state, l.mode
FROM pg_stat_activity a
JOIN pg_locks l ON a.pid = l.pid
WHERE NOT l.granted;
```

Para ver que sesion esta bloqueando a otra, se puede consultar `pg_blocking_pids()`:

```sql
SELECT pid, pg_blocking_pids(pid) AS bloqueado_por, query
FROM pg_stat_activity
WHERE state = 'active'
  AND pid <> pg_backend_pid();
```

Con estas herramientas un administrador puede identificar la sesion bloqueante y, si es necesario, terminarla con `pg_terminate_backend(pid)` para liberar los recursos retenidos.

> Terminar una transaccion con `pg_terminate_backend` fuerza un rollback de todo el trabajo no confirmado. Es una medida de ultimo recurso que debe emplearse con cautela, especialmente en transacciones largas que hayan acumulado mucho trabajo.

