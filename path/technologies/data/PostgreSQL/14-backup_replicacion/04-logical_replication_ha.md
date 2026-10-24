# Logical Replication y High Availability

La replicacion logica introduce un nivel de abstraccion sobre el WAL que permite replicar datos a nivel de tabla entre servidores independientes, incluso con versiones distintas de PostgreSQL o entre bases de datos que solo comparten un subconjunto de su esquema. Combinada con herramientas de alta disponibilidad, forma la base de las arquitecturas PostgreSQL tolerantes a fallos en entornos de produccion.

## Logical Replication con Publication/Subscription

A diferencia de la replicacion fisica, que opera a nivel de cluster completo y requiere que ambos servidores tengan exactamente la misma estructura, la replicacion logica replica cambios fila a fila utilizando un mecanismo de publicacion-suscripcion inspirado en el patron observer.

### Publicacion (Publisher)

El servidor que origina los datos define una o mas publicaciones que agrupan las tablas a replicar:

```sql
-- Publicacion de todas las tablas en la base de datos
CREATE PUBLICATION mi_pub FOR ALL TABLES;

-- Publicacion de tablas especificas
CREATE PUBLICATION pedidos_pub FOR TABLE pedidos, detalle_pedido;

-- Publicacion con filtro de filas (PG 15+)
CREATE PUBLICATION clientes_activos FOR TABLE clientes WHERE (activo = true);
```

Parametros adicionales:

- `WITH (publish = 'insert, update, delete, truncate')`: controla que operaciones se publican.
- `WITH (publish_via_partition_root = true)`: para tablas particionadas, publica desde la raiz.

### Suscripcion (Subscriber)

El servidor destino crea una suscripcion a una publicacion remota:

```sql
CREATE SUBSCRIPTION mi_sub
CONNECTION 'host=192.168.1.10 port=5432 dbname=mi_bd user=replicador password=secreto'
PUBLICATION mi_pub;
```

La suscripcion, por defecto, copia los datos iniciales de las tablas publicadas y luego aplica los cambios en streaming.

> Es responsabilidad del administrador que el esquema de las tablas suscritas exista en el subscriptor antes de crear la suscripcion. PostgreSQL no replica DDL; solo replica los cambios DML.

### Replicacion entre versiones

La replicacion logica funciona entre versiones distintas de PostgreSQL siempre que ambas sean compatibles con el protocolo de replicacion logica (PG 10 o superior). Esto permite migraciones sin tiempo de inactividad: se replica logicamente a una version superior y, cuando estan sincronizadas, se redirige el trafico.

### pg_stat_subscription

La vista `pg_stat_subscription` muestra el estado de cada suscripcion:

```sql
SELECT subname, pid, received_lsn, last_msg_send_time, last_msg_receipt_time
FROM pg_stat_subscription;
```

Si el proceso apply (`pid`) es NULL, la suscripcion no esta activa. Los motivos pueden ser errores de conexion, conflictos de datos o falta de permisos.

### Limitaciones y consideraciones

- No replica secuencias (cada servidor mantiene su propia secuencia).
- No replica DDL ni objetos grandes.
- Cada tabla replicada necesita una clave primaria o `REPLICA IDENTITY` configurada.
- Los conflictos (misma fila modificada en ambos lados) detienen la replicacion hasta resolverse manualmente.

```sql
-- Configurar REPLICA IDENTITY si la tabla no tiene PK
ALTER TABLE mi_tabla REPLICA IDENTITY FULL;
```

## Patroni: HA para PostgreSQL

Patroni es un sistema de alta disponibilidad que automatiza la gestion de clusters PostgreSQL con replicacion por streaming. Utiliza un almacen externo (Distributed Configuration Store o DCS) como etcd, Consul o ZooKeeper para coordinar el estado del cluster y decidir que nodo es el primario.

### Arquitectura de Patroni

Cada nodo PostgreSQL ejecuta un proceso Patroni que monitorea la salud del servidor y se comunica con el DCS. Cuando el primario falla, los nodos restantes realizan una eleccion:

1. Los standbys detectan la ausencia del primario mediante el DCS.
2. El nodo con el WAL mas avanzado (candidato optimo) se promueve a primario.
3. Los demas standbys se reconfiguran automaticamente para seguir al nuevo primario.

Configuracion tipica de Patroni (`/etc/patroni/patroni.yml`):

```yaml
scope: cluster1
name: node1

restapi:
  listen: 0.0.0.0:8008
  connect_address: 192.168.1.10:8008

etcd:
  host: 192.168.1.100:2379

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 192.168.1.10:5432
  data_dir: /var/lib/postgresql/17/main
  parameters:
    wal_level: replica
    hot_standby: "on"
    max_wal_senders: 5
    max_connections: 200
  authentication:
    replication:
      username: replicador
      password: secreto
    superuser:
      username: postgres
      password: postgres
```

Patroni expone una API REST en el puerto 8008 que permite consultar el estado del cluster, forzar switchover o conocer el nodo primario desde los balanceadores de carga.

## PgBouncer: connection pooling

PgBouncer es un pool de conexiones ligeras que se situa entre las aplicaciones y PostgreSQL. Reduce el overhead de crear y destruir conexiones, permitiendo que cientos o miles de conexiones de aplicacion se multiplexen sobre un numero reducido de conexiones reales a la base de datos.

Modos de operacion:

- **Session pooling**: mas compatible, asigna una conexion de servidor por cada conexion de cliente durante toda la sesion.
- **Transaction pooling**: asigna una conexion solo durante la duracion de una transaccion. Maximo rendimiento pero requiere que las aplicaciones no usen caracteristicas stateful (como `SET` persistente, `LISTEN/NOTIFY`, o cursores abiertos entre transacciones).
- **Statement pooling**: asigna y libera por cada sentencia. Muy restrictivo, util solo para casos muy especificos.

Configuracion basica (`pgbouncer.ini`):

```ini
[databases]
mi_bd = host=127.0.0.1 port=5432 dbname=mi_bd

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
default_pool_size = 25
max_client_conn = 500
```

Las aplicaciones se conectan al puerto de PgBouncer (6432) en lugar del puerto de PostgreSQL (5432).

## Pgpool-II: load balancing y pooling

Pgpool-II ofrece funcionalidades que van mas alla del pooling simple: balanceo de carga de consultas de solo lectura entre standbys, failover automatico basico, y limites de conexion.

En un escenario con replicacion por streaming, Pgpool-II puede:

- Enviar consultas SELECT a cualquier standby mientras las escrituras van al primario.
- Detectar la caida del primario y promover un standby.
- Mantener la sesion del cliente transparente durante el failover si se configura adecuadamente.

La configuracion de balanceo en `pgpool.conf`:

```
backend_hostname0 = '192.168.1.10'
backend_port0 = 5432
backend_weight0 = 0   # primario: sin carga de solo lectura
backend_hostname1 = '192.168.1.11'
backend_port1 = 5432
backend_weight1 = 1   # standby: recibe consultas SELECT
```

> Pgpool-II y PgBouncer no son excluyentes. Una arquitectura comun combina PgBouncer para el pool de conexiones (mas eficiente) y Pgpool-II solo para balanceo de lectura y failover, o bien se utiliza Patroni para la gestion del cluster y PgBouncer para el pooling.

## Failover automatico

El failover automatico es la capacidad de promocionar un standby a primario sin intervencion manual. En PostgreSQL nativo no existe este mecanismo; debe implementarse mediante una herramienta externa:

- **Patroni**: la opcion mas madura y ampliamente adoptada en entornos de produccion.
- **repmgr**: gestion de replicacion con herramientas de failover y switchover.
- **pg_auto_failover**: extension mantenida por Citus (Microsoft) para failover automatico simplificado.
- **pgpool-II**: incluye watchdog para failover basado en votacion entre multiples nodos pgpool.

Independientemente de la herramienta, el failover debe considerar:

1. **Deteccion**: diferenciar un fallo real de una particion de red temporal.
2. **Split-brain prevention**: evitar que dos nodos se consideren primarios simultaneamente (el DCS de Patroni previene esto).
3. **Redireccion de trafico**: actualizar DNS, balanceadores o IP virtual flotante (VIP) para que las aplicaciones encuentren al nuevo primario.
4. **Reincorporacion**: cuando el nodo caido se recupera, debe reincorporarse como standby sin reintroducir datos obsoletos.

