# Herramientas

PostgreSQL incluye un conjunto completo de herramientas nativas para la administracion del servidor y la gestion de datos, ademas de contar con un ecosistema de clientes graficos y soluciones de backup desarrolladas por la comunidad y terceros. La linea de comandos sigue siendo el metodo mas potente y flexible, mientras que las interfaces graficas facilitan la exploracion y el mantenimiento en entornos con menor experiencia tecnica.

## psql

La interfaz de linea de comandos oficial es la herramienta mas versatil para interactuar con PostgreSQL. Se conecta a una base de datos con:

```bash
psql -h localhost -p 5432 -U usuario -d basedatos
```

Una vez dentro del prompt interactivo, los metacomandos permiten explorar el servidor sin escribir SQL:

| Metacomando | Descripcion |
|-------------|-------------|
| `\l` | Lista todas las bases de datos del cluster |
| `\dt` | Muestra las tablas del esquema actual |
| `\d tabla` | Describe una tabla (columnas, tipos, restricciones, indices) |
| `\du` | Lista los roles del servidor |
| `\dx` | Enumera las extensiones instaladas |
| `\x` | Activa o desactiva la visualizacion expandida |
| `\c bd` | Conecta a otra base de datos |
| `\conninfo` | Muestra informacion de la conexion actual |
| `\q` | Sale de psql |
| `\?` | Muestra la ayuda completa de metacomandos |
| `\h comando` | Ayuda sobre un comando SQL especifico |

El metacomando `\x` es especialmente util cuando una fila tiene muchas columnas: alterna entre formato tabular y formato vertical, mostrando cada columna en su propia linea para facilitar la lectura.

```sql
-- Ejecutar consultas SQL directamente desde el prompt
SELECT * FROM pg_stat_activity;

-- Obtener ayuda sobre la sintaxis de un comando
\h SELECT
```

> psql dispone de historial de comandos navegable con las flechas arriba y abajo, autocompletado con TAB, edicion multilinea y busqueda en el historial con Ctrl+R.

## pgAdmin

Herramienta grafica oficial escrita en Python con interfaz web. Se distribuye como aplicacion independiente con un servidor web embebido al que se accede desde el navegador.

```bash
# Instalacion en Ubuntu
sudo apt install pgadmin4

# Iniciar el servidor web
pgadmin4
```

Caracteristicas principales:

- Arbol jerarquico de servidores, bases de datos, esquemas y objetos
- Editor SQL con resaltado de sintaxis y autocompletado
- Visor grafico de planes de ejecucion (EXPLAIN)
- Monitor de conexiones activas y bloqueos
- Herramienta de backup y restore con interfaz grafica
- Dashboard de rendimiento en tiempo real

pgAdmin se conecta a los servidores mediante conexion directa o a traves de un tunel SSH, lo que permite administrar bases de datos remotas sin exponer puertos en el firewall.

## Clientes SQL

Varios clientes SQL de terceros ofrecen soporte nativo para PostgreSQL con interfaces graficas mas pulidas o especializadas que pgAdmin.

### DBeaver

Cliente universal multiplataforma con soporte para PostgreSQL, MySQL, SQL Server y decenas de motores mas. Incluye editor SQL con autocompletado, visor de diagramas entidad-relacion, exportacion de datos a multiples formatos y navegacion completa por esquemas, tablas, vistas y procedimientos.

### DataGrip

IDE de JetBrains especializado en bases de datos. Ofrece refactorizacion de codigo SQL, analisis de rendimiento, navegacion inteligente entre objetos, control de versiones integrado y asistencia completa para PostgreSQL. Es software de pago pero dispone de periodo de prueba.

### Beekeeper Studio

Cliente SQL moderno y ligero disponible en version gratuita (community) y de pago (ultimate). Su interfaz minimalista lo hace ideal para consultas rapidas y exploracion de esquemas. Soporta PostgreSQL, MySQL, SQLite y SQL Server con conexion nativa.

## Herramientas de backup

PostgreSQL proporciona herramientas nativas para backup logico y fisico, ademas de soluciones externas especializadas en entornos de produccion con grandes volumenes de datos.

### pg_dump

Realiza un backup logico de una base de datos. Genera un archivo con las sentencias SQL necesarias para reconstruir la base de datos al momento de la ejecucion. Es la herramienta mas utilizada para backups de bases de datos individuales.

```bash
# Backup en formato texto plano (SQL puro)
pg_dump -U postgres -d basedatos > backup.sql

# Backup en formato custom (comprimido, permite restauracion selectiva)
pg_dump -U postgres -Fc -d basedatos > backup.dump

# Backup en formato tar
pg_dump -U postgres -Ft -d basedatos > backup.tar
```

Opciones mas relevantes:

- `-Fc` / `-Ft` / `-Fp`: formato custom, tar o plain
- `-j N`: numero de trabajos paralelos (solo formato custom)
- `-t tabla`: respaldar solo una tabla especifica
- `--exclude-table`: excluir una tabla del backup
- `-Z 0-9`: nivel de compresion

### pg_dumpall

Similar a `pg_dump` pero respalda **todas las bases de datos** del cluster, incluyendo objetos globales como roles, tablespaces y configuraciones de acceso.

```bash
pg_dumpall -U postgres > backup_completo.sql
```

Para restaurar un backup generado con `pg_dumpall` se utiliza directamente `psql`, ya que el archivo contiene tanto los comandos de creacion de bases de datos como los datos:

```bash
psql -U postgres -f backup_completo.sql
```

### pg_restore

Restaura backups creados con `pg_dump` en formato custom o tar. Permite seleccionar que objetos restaurar, en que orden y con que nivel de paralelismo.

```bash
# Restaurar un backup custom completo
pg_restore -U postgres -d basedatos backup.dump

# Restaurar solo una tabla especifica
pg_restore -U postgres -d basedatos -t tabla backup.dump

# Restaurar con 4 trabajos paralelos
pg_restore -U postgres -d basedatos -j 4 backup.dump
```

Opciones clave:

- `-d basedatos`: base de datos destino
- `-t tabla`: restaurar solo una tabla
- `-j N`: numero de trabajos paralelos
- `--clean`: eliminar objetos existentes antes de restaurar
- `--if-exists`: anadir `IF EXISTS` a los comandos DROP

### pg_basebackup

Genera un backup fisico (copia a nivel de archivos) del cluster completo. Es la base para configurar replicacion en streaming y para backups utilizados en point-in-time recovery.

```bash
pg_basebackup -U postgres -D /backup/dir -Fp -Xs -P
```

Opciones principales:

- `-D directorio`: directorio de salida del backup
- `-Fp` / `-Ft`: formato plain (directorio) o tar
- `-Xs`: incluir el WAL en modo streaming
- `-P`: mostrar progreso de la operacion
- `-R`: crear los archivos de configuracion para replicacion (standby.signal)

### pgBackRest

Herramienta de backup y restore disenada para entornos de produccion con grandes volumenes de datos. Soporta backups completos, incrementales y diferenciales, compresion, cifrado y paralelismo.

```bash
# Backup completo
pgbackrest --stanza=main --type=full backup

# Restaurar al estado mas reciente
pgbackrest --stanza=main restore

# Listar backups disponibles
pgbackrest --stanza=main info
```

pgBackRest puede almacenar backups en disco local, NFS, S3, Azure Blob y GCS. Su arquitectura por capas permite realizar backups en caliente sin interrumpir el servicio y gestiona politicas de retencion automaticas.

### Barman

Backup and Recovery Manager, desarrollado por EnterpriseDB. Gestiona backups remotos de multiples servidores PostgreSQL con soporte para politicas de retencion, WAL streaming y recuperacion point-in-time.

```bash
# Backup completo de un servidor
barman backup servidor_produccion

# Listar backups disponibles
barman list-backup servidor_produccion

# Restaurar al punto mas reciente
barman recover servidor_produccion /ruta/restauracion

# Recuperacion point-in-time a una fecha y hora concretas
barman recover servidor_produccion /ruta/restauracion --target-time "2026-07-28 12:00:00"
```

Barman se integra con sistemas de monitoreo, envia notificaciones por correo electronico y permite programar politicas de retencion automaticas. Es la opcion recomendada para entornos empresariales que requieren gestion centralizada de backups de PostgreSQL.

