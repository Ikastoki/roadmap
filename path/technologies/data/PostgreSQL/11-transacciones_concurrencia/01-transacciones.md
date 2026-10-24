# Transacciones en PostgreSQL

Una transaccion es la unidad logica de trabajo que agrupa una o varias operaciones sobre la base de datos como un todo indivisible. PostgreSQL implementa el modelo transaccional completo del estandar SQL, donde cada transaccion garantiza las propiedades ACID (Atomicity, Consistency, Isolation, Durability). A diferencia de motores que operan en modo autocommit implicito sin posibilidad de desactivarlo, PostgreSQL otorga al desarrollador control explicito sobre los limites de cada transaccion mediante las sentencias `BEGIN`, `COMMIT` y `ROLLBACK`, ademas de permitir puntos de restauracion intermedios con `SAVEPOINT`. Este control fino es la base sobre la que se construyen sistemas que requieren consistencia en entornos con escritura concurrente, desde operaciones bancarias hasta pipelines de ingesta de datos que deben ejecutarse completamente o no ejecutarse en absoluto.

## ACID: atomicidad, consistencia, aislamiento y durabilidad

Las siglas ACID definen las garantias que debe proporcionar un sistema transaccional. La atomicidad asegura que todas las operaciones dentro de una transaccion se completan o ninguna se aplica; si el servidor se detiene a mitad de una transaccion, al reiniciar solo persisten los cambios confirmados. La consistencia garantiza que cada transaccion lleva la base de datos de un estado valido a otro estado valido, respetando restricciones, disparadores y reglas definidas en el esquema. El aislamiento protege a cada transaccion de ver los cambios intermedios de otras transacciones concurrentes; el nivel concreto se configura mediante `SET TRANSACTION ISOLATION LEVEL`. La durabilidad asegura que una vez ejecutado `COMMIT`, los cambios sobreviven a cualquier fallo del sistema, gracias al Write-Ahead Log (WAL) que registra cada modificacion en disco antes de confirmarla.

> PostgreSQL no compromete ninguna de las cuatro propiedades en su configuracion por defecto. Sin embargo, parametros como `synchronous_commit` pueden relajar la durabilidad a cambio de rendimiento, permitiendo que el `COMMIT` retorne antes de que el WAL se haya escrito fisicamente en disco.

## BEGIN, COMMIT y ROLLBACK

Una transaccion se inicia con `BEGIN` (o sus equivalentes `START TRANSACTION` y `BEGIN WORK`). Todas las sentencias ejecutadas a continuacion operan dentro del mismo contexto transaccional hasta que se encuentra un `COMMIT`, que hace permanentes los cambios, o un `ROLLBACK`, que los deshace por completo.

```sql
BEGIN;
UPDATE cuentas SET saldo = saldo - 100 WHERE id = 1;
UPDATE cuentas SET saldo = saldo + 100 WHERE id = 2;
COMMIT;
```

Si la segunda actualizacion falla (por ejemplo, porque la cuenta destino no existe), el `ROLLBACK` revierte ambas operaciones y la base de datos queda en el estado anterior al `BEGIN`.

```sql
BEGIN;
UPDATE cuentas SET saldo = saldo - 100 WHERE id = 1;
UPDATE cuentas SET saldo = saldo + 100 WHERE id = 999;  -- falla
ROLLBACK;  -- la cuenta 1 recupera su saldo original
```

PostgreSQL gestiona internamente el estado de cada transaccion mediante un identificador de transaccion (xid) que se asigna en el momento en que la transaccion realiza su primera operacion de escritura. Hasta ese momento, la transaccion no tiene xid y solo lee datos, lo que evita consumir xids innecesariamente.

## Modo autocommit

Por defecto, en la mayoria de clientes interactivos como `psql`, PostgreSQL opera en modo autocommit: cada sentencia ejecutada fuera de un bloque `BEGIN ... COMMIT` se envuelve automaticamente en una transaccion implicita que se confirma al terminar la sentencia. Si una sentencia falla, el autocommit ejecuta un `ROLLBACK` implicito, pero solo para esa sentencia.

Este comportamiento puede desactivarse con `\set AUTOCOMMIT off` en psql, o desde la aplicacion configurando la conexion para que no autoconfirme. Cuando el autocommit esta desactivado, toda operacion queda pendiente hasta que la aplicacion emita explicitamente `COMMIT` o `ROLLBACK`.

> En entornos de produccion es habitual desactivar el autocommit desde el pool de conexiones para tener control explicito sobre los limites transaccionales y evitar que una secuencia de operaciones que deberia ser atomica se confirme parcialmente.

## SAVEPOINT y rollback parcial

Un `SAVEPOINT` define un punto intermedio dentro de una transaccion al que se puede retroceder sin abortar toda la transaccion. Esto permite manejar errores de forma granular: se intenta una operacion, si falla se revierte solo hasta el savepoint y se continua con el resto del trabajo.

```sql
BEGIN;
INSERT INTO bitacora VALUES ('inicio proceso');
SAVEPOINT sp_procesar;
INSERT INTO datos_validados SELECT * FROM temporal WHERE valido = true;
-- Si algo falla, retrocedemos al savepoint
ROLLBACK TO SAVEPOINT sp_procesar;
-- Registramos el error y continuamos
INSERT INTO errores VALUES ('fallo en validacion');
-- Liberamos el savepoint (opcional)
RELEASE SAVEPOINT sp_procesar;
COMMIT;
```

Los savepoints se pueden anidar. Cada nuevo savepoint con el mismo nombre reemplaza al anterior. La sentencia `RELEASE SAVEPOINT` elimina el savepoint sin modificar los datos; es buena practica liberarlos cuando ya no se necesitan para liberar recursos internos del motor.

> El uso de savepoints tiene un costo: cada savepoint retiene recursos hasta que se libera o hasta que termina la transaccion. En transacciones muy largas con cientos de savepoints, el rendimiento puede degradarse.

## Transacciones anidadas y limitaciones

PostgreSQL no soporta transacciones anidadas en el sentido del estandar SQL (no existe `BEGIN` dentro de otra transaccion que espere a un `COMMIT` interno). Sin embargo, los savepoints proporcionan una funcionalidad equivalente en la practica: el comportamiento de `ROLLBACK TO SAVEPOINT` es identico al de un rollback parcial en sistemas que si soportan anidamiento real.

