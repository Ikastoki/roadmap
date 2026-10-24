# Seguridad y mantenimiento

La administracion de una base de datos PostgreSQL no termina con el despliegue inicial. La seguridad exige un modelo de privilegios bien definido, cifrado en las capas adecuadas y auditoria continua. El mantenimiento, por su parte, requiere la ejecucion sistematica de tareas que preservan la salud del motor y la integridad de los datos.

## Principio de minimo privilegio

Cada rol debe tener exclusivamente los permisos necesarios para realizar su funcion, y nada mas. Nunca se debe usar el superusuario `postgres` para operaciones cotidianas de la aplicacion.

```sql
-- Rol para la aplicacion web (solo necesita CRUD basico)
CREATE ROLE app_web WITH LOGIN PASSWORD 'segura123';
GRANT CONNECT ON DATABASE mibd TO app_web;
GRANT USAGE ON SCHEMA public TO app_web;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_web;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_web;

-- Rol para el equipo de analisis (solo lectura)
CREATE ROLE analista WITH LOGIN PASSWORD 'analisis456';
GRANT CONNECT ON DATABASE mibd TO analista;
GRANT USAGE ON SCHEMA public TO analista;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO analista;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT ON TABLES TO analista;

-- Rol para DBA (administracion, no aplicacion)
CREATE ROLE dba WITH LOGIN PASSWORD 'dba789' SUPERUSER;
```

Los permisos a nivel de columna y fila permiten un control aun mas granular:

```sql
-- Solo visible para rol nominas
CREATE VIEW empleados_nomina AS
    SELECT id, salario, numero_cuenta, iban
    FROM empleados;

GRANT SELECT ON empleados_nomina TO rol_nominas;

-- Row-Level Security: filtrar filas por criterio
ALTER TABLE pedidos ENABLE ROW LEVEL SECURITY;
CREATE POLICY pedidos_region_policy ON pedidos
    USING (region = current_setting('app.region'));
GRANT SELECT ON pedidos TO app_web;
```

## Roles dedicados (no postgres)

El superusuario `postgres` debe reservarse exclusivamente para tareas de administracion global: creacion de bases de datos, gestion de roles, configuracion del servidor. Para operaciones rutinarias se definen roles funcionales:

| Rol | Uso |
|-----|-----|
| `app_web` | Conexion desde el backend de la aplicacion |
| `app_readonly` | Consultas de reporting y BI |
| `dba` | Administracion, backup, monitoreo |
| `migrator` | Ejecucion de migraciones de esquema (DDL) |

```sql
CREATE ROLE migrator WITH LOGIN PASSWORD 'migraciones';
GRANT ALL PRIVILEGES ON DATABASE mibd TO migrator;
-- Dar permisos de creacion en el esquema solo durante ventanas de migracion
GRANT CREATE ON SCHEMA public TO migrator;
-- Revocar tras la migracion:
REVOKE CREATE ON SCHEMA public FROM migrator;
```

## SSL y cifrado de conexiones

PostgreSQL soporta cifrado TLS nativo para las conexiones cliente-servidor. La configuracion se realiza en `postgresql.conf` y `pg_hba.conf`.

```ini
# postgresql.conf
ssl = on
ssl_cert_file = '/etc/ssl/certs/server.crt'
ssl_key_file = '/etc/ssl/private/server.key'
ssl_ca_file = '/etc/ssl/certs/ca.crt'
ssl_min_protocol_version = 'TLSv1.3'
```

```ini
# pg_hba.conf - solo conexiones SSL para usuarios remotos
hostssl mibd app_web 10.0.0.0/8 scram-sha-256
hostssl mibd analista 10.0.0.0/8 scram-sha-256
```

Para cifrar datos sensibles en reposo, PostgreSQL ofrece el modulo `pgcrypto` que permite cifrado a nivel de aplicacion dentro de la base de datos:

```sql
CREATE EXTENSION pgcrypto;

-- Insertar con cifrado simetrico
INSERT INTO tarjetas (numero_cifrado) VALUES (
    pgp_sym_encrypt('4532756289012345', 'clave_cifrado_segura')
);

-- Leer con descifrado
SELECT pgp_sym_decrypt(numero_cifrado, 'clave_cifrado_segura') AS numero
FROM tarjetas;
```

> El cifrado a nivel de columna con `pgcrypto` protege los datos incluso si alguien obtiene acceso fisico al disco o a un volcado de base de datos. Sin embargo, la clave de cifrado debe gestionarse externamente (vault, HSM) y no almacenarse en la propia base de datos.

## Auditoria

El registro de auditoria permite rastrear quien hizo que, cuando y desde donde. El modulo `pgaudit` es la solucion mas completa para PostgreSQL.

```ini
# postgresql.conf
shared_preload_libraries = 'pgaudit'
pgaudit.log = 'write,ddl,role'     -- auditar escrituras, DDL y cambios de rol
pgaudit.log_level = 'notice'
pgaudit.log_relation = on          -- incluir nombre de tablas afectadas
```

Sin `pgaudit`, se puede habilitar el log de sentencias en `postgresql.conf`:

```ini
log_statement = 'mod'              -- log de INSERT, UPDATE, DELETE, DDL
log_line_prefix = '%t %u %d %r '   -- timestamp, usuario, bd, host
```

Para una auditoria a nivel de aplicacion, es util la funcion `current_setting` combinada con variables de sesion personalizadas. La aplicacion establece el identificador de usuario al abrir la conexion:

```sql
SET app.usuario_id = '42';
SET app.sesion_id = 'abc-123-def';

-- Consultar en pg_stat_activity o triggers de auditoria
SELECT current_setting('app.usuario_id', true);
```

## Rotacion de contrasenas

Las contrasenas de los roles deben rotarse periodicamente. PostgreSQL almacena las contrasenas con hash `scram-sha-256` desde la version 10, pero la rotacion sigue siendo responsabilidad del administrador.

```sql
ALTER ROLE app_web PASSWORD 'nueva_contrasena';
-- Verificar metodo de autenticacion en pg_hba.conf
-- Debe usar scram-sha-256, no md5 ni password
```

Para automatizar la rotacion, es practica comun integrar la base de datos con un gestor de secretos externo (HashiCorp Vault, AWS Secrets Manager, Kubernetes Secrets) que actualiza la contrasena y la distribuye a los clientes sin intervencion manual.

## VACUUM ANALYZE regular

PostgreSQL utiliza MVCC (Multi-Version Concurrency Control), lo que significa que las filas actualizadas o eliminadas no se borran fisicamente de la tabla: se marcan como muertas (_dead tuples_). El proceso `VACUUM` recupera ese espacio y lo marca como reusable. `ANALYZE` actualiza las estadisticas que el planificador de consultas usa para elegir los planes de ejecucion.

```sql
-- Ejecucion manual
VACUUM (VERBOSE, ANALYZE) pedidos;

-- Ver estado de una tabla
SELECT relname, n_live_tup, n_dead_tup, last_autovacuum, last_analyze
FROM pg_stat_user_tables
WHERE relname = 'pedidos';
```

PostgreSQL ejecuta `autovacuum` de forma automatica, pero su configuracion por defecto puede no ser adecuada para tablas con alta tasa de escritura. Es recomendable ajustar los parametros por tabla:

```sql
ALTER TABLE pedidos SET (
    autovacuum_vacuum_scale_factor = 0.01,
    autovacuum_analyze_scale_factor = 0.005,
    autovacuum_vacuum_threshold = 1000
);
```

Configuraciones globales relevantes en `postgresql.conf`:

```ini
autovacuum = on
autovacuum_max_workers = 4
autovacuum_naptime = 30s
autovacuum_vacuum_scale_factor = 0.02    ; vaciar cuando el 2% de las filas son muertas
autovacuum_vacuum_threshold = 1000       ; minimo de filas muertas para disparar vacio
```

## Monitorear dead tuples

Un numero elevado de dead tuples sin vaciar degrada el rendimiento de las consultas (el motor debe recorrerlas para determinar visibilidad) y puede causar _bloat_ (crecimiento excesivo de la tabla).

```sql
SELECT relname,
       n_dead_tup,
       n_live_tup,
       round(n_dead_tup * 100.0 / GREATEST(n_live_tup + n_dead_tup, 1), 2) AS dead_pct,
       last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;
```

Si una tabla supera sistematicamente el 20-30% de dead tuples, es necesario ajustar los parametros de autovacuum para esa tabla o revisar el patron de acceso (posibles updates masivos sin vaciado).

## Reconstruir indices

Con el tiempo, los indices B-Tree pueden fragmentarse o contener paginas muertas. La reconstruccion periodica con `REINDEX` recupera espacio y mejora la eficiencia.

```sql
-- Reconstruir un indice concreto
REINDEX INDEX idx_pedidos_cliente_id;

-- Reconstruir todos los indices de una tabla (con lock exclusivo)
REINDEX TABLE pedidos;

-- Reconstruir concurrentemente (sin bloquear escrituras, desde PG12)
REINDEX INDEX CONCURRENTLY idx_pedidos_cliente_id;
```

`REINDEX CONCURRENTLY` evita bloquear la tabla para escrituras durante la reconstruccion, aunque consume mas recursos y tarda mas tiempo. Es la opcion preferida en sistemas productivos.

## Revision de logs

Los logs contienen informacion fundamental para diagnosticar problemas de rendimiento, errores de autenticacion, consultas lentas y violaciones de integridad.

```ini
# postgresql.conf
log_destination = 'csvlog'
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d.log'
log_min_duration_statement = 500       ; log de consultas que tardan > 500ms
log_checkpoints = on
log_connections = on
log_disconnections = on
lock_timeout = 30000                   ; timeout para locks (30s)
log_lock_waits = on                    ; registrar esperas por lock
```

La revision periodica de las consultas lentas (`log_min_duration_statement`) permite identificar los puntos calientes que requieren optimizacion. Herramientas como `pgBadger` analizan los logs y generan informes de rendimiento estructurados.

## Actualizar estadisticas

El planificador de PostgreSQL se basa en estadisticas sobre la distribucion de los datos para estimar el coste de los planes de ejecucion. `ANALYZE` las recolecta automaticamente como parte de `autovacuum`, pero tras cambios masivos (cargas, eliminaciones de gran volumen) conviene forzar la actualizacion:

```sql
ANALYZE (VERBOSE) pedidos;

-- Estadisticas detalladas de una columna
SELECT attname, n_distinct, most_common_vals, correlation
FROM pg_stats
WHERE tablename = 'pedidos' AND attname = 'total';
```

El valor `correlation` indica si los valores fisicos del disco siguen el orden logico de la columna. Una correlacion cercana a 1 o -1 favorece los `Index Scan` en rangos.

## Backup probado

Un backup que no se restaura periodicamente no es un backup: es un archivo muerto. La unica forma de verificar que el proceso de recuperacion funciona es ejecutarlo en un entorno aislado de forma regular.

PostgreSQL ofrece dos mecanismos principales de backup:

**pg_dump / pg_dumpall** - volcado logico a nivel de base de datos:

```bash
pg_dump -h localhost -U dba -Fc --compress=9 -f mibd_$(date +%Y%m%d).dump mibd
pg_restore -h localhost -U dba -d mibd_test --schema-only mibd_20240101.dump
```

**PgBarman** - backup fisico continuo con Point-In-Time Recovery (PITR):

```bash
# Configurar barman (barman.conf)
[mibd]
description = "Base de datos de produccion"
conninfo = host=10.0.0.1 user=barman dbname=mibd
backup_method = rsync
retention_policy = RECOVERY WINDOW OF 30 DAYS

# Realizar y verificar backup
barman backup mibd
barman check mibd
barman list-backup mibd

# Simular restauracion en directorio temporal
barman recover --remote-ssh-command="ssh backup-server" mibd latest /var/lib/postgresql/restore
```

La estrategia de backup debe contemplar:

1. Backups logicos diarios (pg_dump) para restauracion rapida de objetos individuales.
2. Backups fisicos con WAL archiving para PITR (Barman, pgBackRest) con retencion configurada segun el RPO del servicio.
3. Pruebas de restauracion automatizadas al menos una vez al mes, documentando el tiempo de recuperacion (RTO).

