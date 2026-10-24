# Streaming Replication

La replicacion por streaming permite mantener uno o mas servidores secundarios (standbys) que reciben y aplican continuamente los cambios producidos en el servidor primario, transfiriendo los segmentos WAL a traves de la red en tiempo real. Es la base sobre la que se construyen la mayoria de arquitecturas de alta disponibilidad en PostgreSQL.

## Arquitectura y funcionamiento

El servidor primario escribe los cambios en el WAL local. Cada standby establece una conexion de replicacion y recibe los registros WAL a medida que se generan, sin esperar a que se complete un segmento completo. Esto reduce drasticamente el retardo respecto al metodo clasico de archivo y recuperacion periodica.

El proceso involucra tres componentes en el primario:

- **WAL sender**: proceso (`walsender`) que se conecta al standby y transmite los registros WAL.
- **WAL receiver**: proceso en el standby que recibe los datos y los escribe en el WAL local del standby.
- **Startup process**: en el standby, aplica los registros WAL recibidos a los archivos de datos.

En el primario, la vista `pg_stat_replication` muestra el estado de cada conexion de replicacion:

```sql
SELECT application_name, state, sync_state, write_lag, flush_lag, replay_lag
FROM pg_stat_replication;
```

## Configuracion del servidor primario

En `postgresql.conf` del primario:

```
wal_level = replica
max_wal_senders = 5          # minimo: numero de standbys esperados
wal_keep_size = 1024         # WAL retenido en pg_wal (MB) por si el standby se atrasa
```

En `pg_hba.conf` se debe permitir la conexion de replicacion desde los standbys:

```
# TYPE  DATABASE       USER           ADDRESS          METHOD
host    replication    replicador     192.168.1.0/24   scram-sha-256
```

Se necesita un rol con el atributo `REPLICATION` (o superusuario) para autenticar la conexion:

```sql
CREATE ROLE replicador WITH LOGIN REPLICATION PASSWORD 'secreto';
```

## Creacion del standby con pg_basebackup

En el servidor standby, se utiliza `pg_basebackup` para obtener una copia inicial del cluster:

```bash
pg_basebackup -h 192.168.1.10 -U replicador -D /var/lib/postgresql/17/main -P -v -X stream
```

Este comando descarga todos los archivos del cluster y los WAL generados durante la transferencia en modo streaming (`-X stream`).

## Configuracion del servidor standby

En `postgresql.conf` del standby:

```
primary_conninfo = 'host=192.168.1.10 port=5432 user=replicador password=secreto application_name=standby1'
primary_slot_name = 'standby1'   # opcional pero recomendado
hot_standby = on                 # permite consultas de solo lectura en el standby
```

El archivo `standby.signal` en el directorio de datos indica a PostgreSQL que debe arrancar en modo standby perpetuo:

```bash
touch /var/lib/postgresql/17/main/standby.signal
```

> Sin `standby.signal`, PostgreSQL aplicaria todos los WAL disponibles y luego pasaria a modo normal (lectura/escritura), rompiendo la cadena de replicacion.

### Slots de replicacion

Un slot de replicacion (`pg_create_physical_replication_slot`) garantiza que el primario no elimine segmentos WAL que el standby aun no ha consumido:

```sql
SELECT pg_create_physical_replication_slot('standby1');
```

Si un standby se desconecta sin un slot, el primario puede reciclar WAL y dejar al standby sin capacidad de alcanzar el estado actual, obligando a regenerar el standby desde cero.

## Replicacion sincrona vs asincrona

En la replicacion **asincrona** (por defecto), el primario confirma una transaccion en cuanto la escribe en su WAL local, sin esperar a que el standby la reciba. Esto minimiza la latencia de las escrituras pero implica perdida de datos si el primario falla antes de que el standby haya recibido los cambios.

En la replicacion **sincrona**, el primario espera a que al menos un standby confirme haber recibido y escrito el WAL antes de confirmar la transaccion al cliente. Esto garantiza RPO cero (sin perdida de datos) a costa de aumentar la latencia.

Configuracion para replicacion sincrona:

```
synchronous_standby_names = 'FIRST 1 (standby1, standby2)'
```

El nombre entre parentesis debe coincidir con el `application_name` configurado en `primary_conninfo` del standby. Con `FIRST 1`, solo el primer standby de la lista en confirmar es necesario para considerar la transaccion como confirmada.

### Cascading replication

Un standby puede actuar a su vez como fuente de replicacion para otros standbys, formando una topologia en arbol. Esto reduce la carga en el primario y permite distribuir la replicacion geograficamente.

En el standby intermedio, se configura `standby.signal` y ademas se habilita `max_wal_senders` para servir a sus propios hijos.

## Failover y switchover

**Failover** es la promocion automatica de un standby a primario cuando el primario original falla. Se realiza con:

```bash
pg_ctl promote -D /var/lib/postgresql/17/main
```

O mediante la funcion:

```sql
SELECT pg_promote();
```

Despues de la promocion, el antiguo primario queda aislado y los demas standbys deben reconfigurarse para seguir al nuevo primario.

**Switchover** (o graceful switch) es la promocion planificada del standby manteniendo la integridad del sistema, tipicamente para tareas de mantenimiento. Se degrada el primario a standby y se promueve el standby deseado, sin perdida de datos.

> El failover no es automatico por si solo en PostgreSQL nativo. Herramientas como Patroni, repmgr o pg_auto_failover anaden la capa de deteccion de fallos y promocion automatizada que PostgreSQL no incluye de serie.

## Monitoreo con pg_stat_replication

La vista `pg_stat_replication` es la fuente principal para monitorizar el estado de la replicacion:

```sql
SELECT application_name,
       state,
       sync_state,
       pg_wal_lsn_diff(pg_current_wal_lsn(), write_lsn) AS write_lag_bytes,
       pg_wal_lsn_diff(pg_current_wal_lsn(), flush_lsn) AS flush_lag_bytes,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS replay_lag_bytes
FROM pg_stat_replication;
```

- `write_lag`: WAL enviado pero no escrito en el standby
- `flush_lag`: WAL escrito pero no aplicado
- `replay_lag`: WAL aplicado (el dato esta visible en el standby si `hot_standby = on`)

En el standby, `pg_stat_wal_receiver` muestra el estado desde su perspectiva:

```sql
SELECT status, receive_start_lsn, received_lsn, last_sender_send_time
FROM pg_stat_wal_receiver;
```

