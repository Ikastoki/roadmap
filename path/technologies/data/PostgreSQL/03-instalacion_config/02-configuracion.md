# Configuracion

PostgreSQL se configura mediante archivos de texto plano ubicados en el directorio de datos del cluster. La mayoria de los parametros se pueden modificar en caliente con `pg_ctl reload` o `SELECT pg_reload_conf()`, aunque algunos requieren reinicio del servidor. Comprender la funcion de cada archivo y cada parametro es esencial para ajustar el comportamiento del motor a las necesidades del hardware y la carga de trabajo.

## Archivos de configuracion

El directorio de datos (`/var/lib/postgresql/16/main/` en Debian, `/var/lib/pgsql/16/data/` en RHEL) contiene tres archivos fundamentales que gobiernan el comportamiento del servidor.

### postgresql.conf

Es el archivo de configuracion principal. Contiene todos los parametros del servidor organizados por secciones: conexiones y autenticacion, memoria, WAL, planificador, registro, estadisticas, etc. Cada linea sigue el formato `parametro = valor`.

```bash
sudo nano /etc/postgresql/16/main/postgresql.conf
```

Los valores booleanos aceptan `on`/`off`, `true`/`false`, `yes`/`no`. Las unidades de memoria se expresan con sufijos como `kB`, `MB`, `GB`, `TB`.

```ini
listen_addresses = 'localhost'
port = 5432
max_connections = 100
shared_buffers = 1GB
wal_level = replica
```

### pg_hba.conf

Controla que clientes pueden conectarse al servidor, desde que direcciones IP, con que metodo de autenticacion y a que bases de datos. Las reglas se evaluan en orden descendente; la primera que coincide con la solicitud de conexion se aplica.

```bash
sudo nano /etc/postgresql/16/main/pg_hba.conf
```

Formato de cada linea:

```
# TYPE   DATABASE   USER        ADDRESS        METHOD
local    all        postgres                   peer
local    all        all                        scram-sha-256
host     all        all        127.0.0.1/32    scram-sha-256
host     all        all        ::1/128         scram-sha-256
```

Los metodos de autenticacion mas utilizados son:

- `trust`: permite la conexion sin contrasena, util solo en redes aisladas
- `peer`: confia en el usuario del sistema operativo (conexiones locales)
- `md5` / `scram-sha-256`: autenticacion con contrasena, siendo `scram-sha-256` el metodo recomendado por seguridad
- `reject`: deniega la conexion explicitamente

> La regla `local all postgres peer` permite al usuario `postgres` del sistema conectarse sin contrasena a traves del socket local. Para habilitar conexiones remotas seguras se debe anadir una linea con la red de origen y `scram-sha-256`.

### postgresql.auto.conf

Archivo generado automaticamente por el comando `ALTER SYSTEM`. Tiene prioridad sobre `postgresql.conf` pero menor que las variables de entorno y los parametros pasados por linea de comandos al arrancar el servidor.

```sql
ALTER SYSTEM SET max_connections = 200;
ALTER SYSTEM SET shared_buffers = '2GB';
```

Los cambios se guardan en `postgresql.auto.conf` y se aplican tras recargar la configuracion. Este archivo no debe editarse manualmente, ya que PostgreSQL lo sobrescribe en cada ejecucion de `ALTER SYSTEM`.

## Puerto y conexiones

Tres parametros determinan como el servidor acepta conexiones entrantes:

```ini
listen_addresses = 'localhost'
port = 5432
max_connections = 100
```

- **listen_addresses**: direcciones IP en las que el servidor escucha. `'*'` para todas las interfaces, `'0.0.0.0'` para solo IPv4, `'localhost'` para conexiones locales unicamente.
- **port**: puerto TCP en el que el servidor acepta conexiones. Por defecto es el 5432, numero registrado oficialmente para PostgreSQL ante IANA.
- **max_connections**: numero maximo de conexiones simultaneas. Cada conexion consume entre 2 y 10 MB de RAM, por lo que valores muy altos requieren suficiente memoria disponible.

## Memoria

La asignacion de memoria es el factor mas determinante en el rendimiento de PostgreSQL. Los cuatro parametros principales deben ajustarse segun la RAM total del servidor y el tipo de carga de trabajo.

### shared_buffers

Cache compartida donde PostgreSQL almacena paginas de datos en memoria. Se recomienda entre el 15 % y el 25 % de la RAM total. Valores superiores al 25 % pueden provocar contencion en el cache y no mejoran el rendimiento debido a la doble gestion de cache con el sistema operativo.

```ini
shared_buffers = 2GB   # Para un servidor con 8 GB de RAM
```

### work_mem

Memoria disponible para operaciones de ordenacion, hash, joins y agregaciones. No es un valor global, sino que se asigna **por consulta y por operacion**: una misma consulta con varios nodos de ordenacion puede consumir multiples veces este valor.

```ini
work_mem = 16MB   # 4-64 MB segun la carga de trabajo
```

Un valor demasiado alto puede agotar la memoria si se ejecutan muchas consultas concurrentes que utilizan ordenaciones o hash joins. Un valor demasiado bajo provoca derrames a disco temporales que degradan el rendimiento.

### maintenance_work_mem

Memoria destinada a operaciones de mantenimiento como `VACUUM`, `CREATE INDEX`, `REINDEX` y `ALTER TABLE ADD FOREIGN KEY`. Al no ejecutarse concurrentemente como las consultas normales, puede ser bastante mas alto que `work_mem`.

```ini
maintenance_work_mem = 256MB   # 64-1024 MB
```

Un valor generoso acelera significativamente las operaciones de mantenimiento, especialmente la creacion de indices en tablas grandes.

### effective_cache_size

Estimacion de la memoria disponible para cache de archivos del sistema operativo. No reserva memoria fisica, sino que ayuda al planificador de consultas a decidir si utilizar un index scan o un sequential scan.

```ini
effective_cache_size = 6GB   # 50-75 % de la RAM total
```

Cuanto mayor sea este valor, mas probable es que el planificador prefiera index scans sobre sequential scans. En servidores dedicados exclusivamente a PostgreSQL se recomienda fijarlo en el 75 % de la RAM total.

## Disco

Los parametros de disco controlan la escritura del Write-Ahead Log (WAL) y como PostgreSQL interactua con el subsistema de entrada/salida.

### wal_level

Define cuanta informacion se escribe en el WAL. Los niveles disponibles son:

- `minimal`: solo lo necesario para recuperacion tras caida
- `replica`: incluye informacion suficiente para replicacion en caliente
- `logical`: anade informacion para decodificacion logica y replicacion logica

```ini
wal_level = replica
```

Para entornos con replicacion o backup continuo basado en WAL archiving es obligatorio usar al menos `replica`.

### max_wal_size

Tamaño maximo total acumulado de los archivos WAL antes de forzar un checkpoint. Si el WAL crece demasiado, la recuperacion tras una caida tarda mas porque el servidor debe replayar todos los segmentos desde el ultimo checkpoint.

```ini
max_wal_size = 4GB   # Valor recomendado: 2-8 GB segun la actividad de escritura
```

### checkpoint_completion_target

Fraccion del intervalo entre checkpoints durante la cual se deben escribir los buffers sucios al disco. Un valor alto distribuye la escritura a lo largo del intervalo, reduciendo los picos de E/S.

```ini
checkpoint_completion_target = 0.9   # 0.5-0.9
```

Con 0.9, cuando ocurre un checkpoint el sistema intenta completar la escritura de todos los buffers sucios en el 90 % del tiempo disponible antes del siguiente checkpoint. Esto suaviza la carga de escritura pero incrementa la probabilidad de perdida de datos en caso de caida.

### random_page_cost

Costo estimado de una lectura aleatoria de disco, expresado en unidades relativas al costo de una lectura secuencial (1.0).

```ini
random_page_cost = 4.0   # HDD tradicional
random_page_cost = 1.1   # SSD
```

En servidores con almacenamiento SSD, reducir este valor a 1.1 hace que el planificador prefiera index scans sobre sequential scans, reflejando mejor el rendimiento real del hardware.

### effective_io_concurrency

Numero de operaciones de E/S simultaneas que el subsistema de disco puede manejar. PostgreSQL utiliza este parametro para decidir cuando lanzar lecturas anticipadas en paralelo.

```ini
effective_io_concurrency = 200   # SSD moderno
effective_io_concurrency = 2     # HDD
```

Los discos SSD se benefician de valores altos (100-300), mientras que los HDD apenas ganan con valores superiores a 2.

> Tras modificar cualquier parametro, ejecutar `SELECT pg_reload_conf();` como superusuario o enviar `pg_ctl reload` para aplicar los cambios sin reiniciar. Los cambios en `shared_buffers`, `wal_level`, `max_connections` y `port` requieren reinicio completo del servidor.

