# Restauracion y Point-in-Time Recovery

La restauracion es el proceso inverso al backup: reconstruir una base de datos a partir de los archivos de respaldo. La complejidad de este proceso depende del tipo de backup disponible y del punto en la historia al que se quiera regresar.

## pg_restore: desde backup logico en formato custom o tar

`pg_restore` es la herramienta para restaurar backups generados con `pg_dump` en formato custom (`-Fc`) o tar (`-Ft`). No funciona con archivos en texto plano; esos se restauran con `psql`.

```bash
pg_restore -U postgres -d mi_bd -v mi_bd.backup
```

Opciones esenciales:

- `-d`: base de datos destino
- `-j n`: paralelismo en restauracion (varios hilos)
- `-t tabla`: restaurar solo una tabla especifica
- `-s`: solo esquema, sin datos
- `-a`: solo datos, sin esquema
- `-c`: eliminar objetos antes de crearlos (clean)
- `--if-exists`: anade IF EXISTS a los comandos DROP

La restauracion selectiva es una de las ventajas del formato custom: permite extraer una tabla concreta de un backup completo sin tener que restaurar todo el cluster:

```bash
pg_restore -U postgres -d mi_bd -t usuarios -t pedidos mi_bd.backup
```

## psql: desde texto plano

Los backups en formato plain (`-Fp`) son archivos SQL que se ejecutan con `psql`:

```bash
psql -U postgres -d mi_bd < backup.sql
```

O alternativamente:

```bash
psql -U postgres -f backup.sql
```

Para bases de datos grandes, conviene desactivar ciertas comprobaciones durante la restauracion para acelerar el proceso:

```bash
psql -U postgres -d mi_bd -f backup.sql -X -v ON_ERROR_STOP=1
```

Y ajustar parametros de sesion al inicio del archivo SQL o mediante variables de entorno (como `PGOPTIONS='-c statement_timeout=0'`).

## Point-in-Time Recovery (PITR)

PITR permite restaurar el cluster a un estado especifico en el tiempo, no solo al momento exacto del ultimo backup. Es posible gracias a la combinacion de un backup fisico (basebackup) mas el archivo continuo de segmentos WAL.

### Proceso de restauracion PITR

1. Detener el servidor PostgreSQL si esta en ejecucion.
2. Restaurar el contenido del basebackup en el directorio de datos destino.
3. Configurar los parametros de restauracion en `postgresql.conf` y crear el archivo senial.
4. Iniciar el servidor: PostgreSQL aplicara todos los WAL archivados hasta alcanzar el punto objetivo.

A partir de PostgreSQL 12, la configuracion se realiza mediante:

- `restore_command` en `postgresql.conf`: comando para recuperar un segmento WAL archivado.
- Archivo `recovery.signal` o `standby.signal` en el directorio de datos: indica al servidor que debe entrar en modo recovery.

```
# postgresql.conf
restore_command = 'cp /backups/wal/%f %p'
recovery_target_time = '2026-07-28 14:30:00 UTC'
```

```bash
# Crear archivo senial de recovery
touch /var/lib/postgresql/17/main/recovery.signal
```

Opciones de objetivo de recuperacion:

- `recovery_target_time`: momento exacto en el tiempo
- `recovery_target_xid`: identificador de transaccion
- `recovery_target_lsn`: posicion exacta en el WAL (a partir de PG 10)
- `recovery_target_name`: nombre definido con `pg_create_restore_point()`
- `recovery_target_timeline`: timeline historico (para saltar entre ramas)

### recovery.conf y postgresql.auto.conf

En versiones anteriores a PostgreSQL 12, los parametros de recovery se colocaban en un archivo `recovery.conf` separado. Desde PG 12, estos parametros se integran en `postgresql.conf` o `postgresql.auto.conf`. La presencia de `recovery.signal` (recovery normal) o `standby.signal` (modo standby perpetuo) determina en que modo arranca el servidor.

> Es critico entender que PITR avanza el cluster hasta el objetivo y luego detiene la recuperacion. Si se necesita continuar aplicando WAL (modo standby), se usa `recovery_target = 'immediate'` o simplemente no se especifica objetivo, lo que hara que el servidor aplique todos los WAL disponibles y luego se quede como standby o avance a modo normal segun la senial presente.

## WAL Archiving en detalle

Write-Ahead Logging (WAL) es el mecanismo por el cual PostgreSQL garantiza la durabilidad de las transacciones: cualquier cambio en los datos se escribe primero en el log WAL y luego se aplica a las tablas. Esto permite que el servidor, tras un fallo, rehaga las transacciones confirmadas reproduciendo los registros WAL desde el ultimo checkpoint.

Para que el WAL sirva en un escenario de PITR, debe estar archivado mas alla del directorio de datos:

```
# postgresql.conf
wal_level = replica         # nivel minimo para archiving
archive_mode = on           # activa el archivado
archive_command = 'aws s3 cp %p s3://mi-bucket-wal/%f'  # comando de archivado
archive_timeout = 300       # fuerza archivado cada 5 minutos si hay poca actividad
```

Cada segmento WAL mide 16 MB por defecto (`wal_segment_size`, configurable en compilacion). Cuando se completa, PostgreSQL ejecuta `archive_command`. Si el comando falla (codigo de salida distinto de cero), el servidor reintenta hasta que tenga exito, acumulando WAL en `pg_wal/` mientras tanto.

Si `pg_wal/` se llena porque el archivado falla durante un periodo prolongado, PostgreSQL se detiene con un error PANIC. Es responsabilidad del operador monitorizar el estado del archivado mediante `pg_stat_archiver`:

```sql
SELECT archived_count, failed_count, last_archived_wal, last_failed_wal
FROM pg_stat_archiver;
```

## pg_waldump: inspeccion del WAL

`pg_waldump` es una herramienta de diagnostico que muestra el contenido de los segmentos WAL en formato legible:

```bash
pg_waldump /ruta/a/pg_wal/0000000100000000000000A1
```

Permite ver que transacciones modificaron que bloques, lo que resulta util para depurar problemas de recuperacion, analizar conflictos de replicacion logica o entender el patron de escritura de la aplicacion.

