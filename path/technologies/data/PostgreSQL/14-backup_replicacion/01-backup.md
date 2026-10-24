# Backup en PostgreSQL

Un backup es una copia de los datos que permite restaurar el sistema ante fallos, errores humanos o desastres. PostgreSQL ofrece dos categorias principales de backup: logicos (a nivel de base de datos con `pg_dump` y `pg_dumpall`) y fisicos (a nivel de archivos del cluster con `pg_basebackup` y WAL archiving). La eleccion entre uno u otro depende de los requisitos de RTO (Recovery Time Objective) y RPO (Recovery Point Objective) del sistema.

## pg_dump: backup logico por base de datos

`pg_dump` extrae una base de datos en un archivo que contiene las sentencias SQL necesarias para reconstruirla, junto con los datos en el formato elegido. No bloquea la base de datos para lecturas, y las operaciones de escritura concurrentes son seguras porque trabaja sobre una snapshot consistente al momento de iniciar.

```bash
pg_dump -U usuario -h localhost -d mi_bd -f backup.sql
```

`pg_dump` exporta el esquema, los datos, las funciones, los tipos definidos por el usuario y otros objetos, pero **no** incluye objetos globales como roles de base de datos, tablespaces o configuraciones de conexion. Para eso existe `pg_dumpall`.

### Formatos de salida

El parametro `-F` (o `--format`) controla el formato del archivo generado:

- **Plain (`-Fp`)**: genera un archivo de texto plano con sentencias SQL. Es el formato por defecto. Se restaura directamente con `psql`. No soporta compresion ni restauracion selectiva.

- **Custom (`-Fc`)**: formato comprimido y enriquecido que permite restauracion selectiva de objetos (tablas, funciones, etc.), paralelismo en restauracion (`-j`) y compresion integrada. Es el formato recomendado para backups de produccion.

- **Tar (`-Ft`)**: genera un archivo tar que contiene multiples archivos internos. Permite restauracion selectiva pero no soporta paralelismo. Es util cuando se quiere inspeccionar el contenido con herramientas de sistema.

```bash
pg_dump -U postgres -Fc -Z 9 -f mi_bd.backup mi_bd
pg_dump -U postgres -Ft -f mi_bd.tar mi_bd
```

El nivel de compresion `-Z` va de 0 (sin compresion) a 9 (maxima). En formato custom, la compresion es por defecto de nivel medio si no se especifica.

### pg_dumpall: backup completo del cluster

`pg_dumpall` realiza un backup logico de todas las bases de datos del cluster e incluye los objetos globales: roles, tablespaces, y configuraciones a nivel de servidor.

```bash
pg_dumpall -U postgres -f cluster_completo.sql
```

El resultado es siempre texto plano (SQL). Para bases de datos muy grandes, una estrategia comun es usar `pg_dumpall --globals-only` para extraer solo los objetos globales, y luego hacer backups individuales por base de datos en formato custom.

## pg_basebackup: backup fisico

`pg_basebackup` crea una copia fisica (a nivel de archivos del cluster) del servidor. Es la herramienta base para configurar replicacion por streaming y para backups que serviran en un point-in-time recovery (PITR).

```bash
pg_basebackup -U replicador -h servidor_maestro -D /backups/base_001 -Fp -P -v
```

Opciones fundamentales:

- `-D`: directorio de destino
- `-Fp` (plain) o `-Ft` (tar): formato del backup
- `-X fetch` o `-X stream`: incluye los WAL generados durante el backup
- `-P`: muestra progreso
- `-l`: etiqueta del backup
- `-z`: comprime (solo con `-Ft`)

Un backup fisico es consistente por si mismo (PostgreSQL garantiza que los archivos copiados representan un estado valido del cluster), pero para poder hacer PITR se necesita ademas el archivo continuo de WAL.

## Backup continuo con WAL archiving

El backup fisico por si solo captura el estado del cluster en un instante. Para poder restaurar a cualquier punto en el tiempo, es necesario archivar continuamente los segmentos WAL que genera el servidor.

La configuracion en `postgresql.conf` requiere:

```
wal_level = replica
archive_mode = on
archive_command = 'cp %p /backups/wal/%f'
```

Cada vez que PostgreSQL completa un segmento WAL (tipicamente 16 MB), ejecuta el comando `archive_command` para copiarlo al destino de archivado. Ese destino puede ser un directorio local, un NFS, o un comando que suba el archivo a S3, GCS o Azure Blob mediante herramientas como `aws s3 cp`, `gsutil cp` o `azcopy`.

El proceso completo de backup continuo implica dos componentes:

1. Un `pg_basebackup` inicial que establece la linea de base (basebackup).
2. El archivado continuo de WAL desde ese punto en adelante.

Para restaurar, se necesita la linea de base mas todos los WAL archivados hasta el momento objetivo.

> El archivado de WAL tiene un retardo inherente: si el servidor falla, se pierden las transacciones confirmadas cuyo WAL aun no ha sido archivado. El RPO minimo depende de la frecuencia con que se completan los segmentos WAL (controlado por `archive_timeout`) o del uso de replicacion sincrona.

## Verificacion de integridad

Ningun backup es util hasta que se prueba su restauracion. Herramientas como `pg_verifybackup` (a partir de PostgreSQL 17) o la practica periodica de restaurar en un entorno de prueba son la unica manera de garantizar que los archivos no estan corruptos o incompletos. Sin verificacion, un backup es simplemente un conjunto de archivos con una falsa sensacion de seguridad.

