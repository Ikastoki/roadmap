# Configuracion de Rendimiento en PostgreSQL

La configuracion por defecto de PostgreSQL esta pensada para entornos con recursos minimos. En un servidor dedicado, ajustar los parametros de memoria y de E/S marca la diferencia entre un sistema que aprovecha el hardware y uno que infrautiliza los recursos disponibles.

Los parametros se definen en `postgresql.conf` o mediante `ALTER SYSTEM SET`. Los cambios requieren recarga (`pg_ctl reload` o `SELECT pg_reload_conf()`) y algunos necesitan reinicio.

## shared_buffers

`shared_buffers` determina la cantidad de memoria dedicada al pool de buffers compartidos, donde PostgreSQL cachea paginas de datos y de indices. Todo acceso a una pagina pasa antes por `shared_buffers`.

El valor recomendado esta entre el 15 % y el 25 % de la RAM total del servidor. Superar el 25 % puede ser contraproducente porque el sistema operativo tambien cachea archivos, y un `shared_buffers` excesivamente grande compite con esa cache del kernel sin ganancia neta.

```ini
# Servidor con 16 GB de RAM
shared_buffers = 4GB
```

En sistemas con mucha RAM (64 GB o mas), el 25 % sigue siendo una referencia segura, aunque algunos workloads pueden beneficiarse de hasta un 30-35 % si el acceso a los datos es muy intensivo.

## work_mem

`work_mem` controla la memoria disponible para operaciones internas de cada consulta: ordenaciones (`ORDER BY`, `DISTINCT`, merge joins), hash tables (`Hash Join`, `Hash Aggregate`) y operaciones de bitmap.

El valor se aplica **por operacion y por consulta**, no por sesion. Una consulta con varios nodos de ordenacion y hash puede consumir multiples veces `work_mem`. Por eso valores muy altos pueden provocar un exceso de memoria agregada.

```ini
work_mem = 16MB
```

El rango tipico esta entre 4 MB y 64 MB. Para determinarlo con precision, se observa en `pg_stat_activity` si aparecen archivos temporales:

```sql
SELECT query, temp_files, temp_bytes
FROM pg_stat_statements
ORDER BY temp_bytes DESC
LIMIT 10;
```

La presencia de `temp_files` indica que `work_mem` es insuficiente para las consultas y PostgreSQL esta derramando a disco, lo cual es drasticamente mas lento.

## effective_cache_size

`effective_cache_size` no reserva memoria, sino que informa al optimizador sobre el tamaño estimado de la cache de archivos del sistema operativo disponible para PostgreSQL. Con este dato, el optimizador puede estimar con mejor precision si un Index Scan (que requiere accesos aleatorios) es preferible a un Seq Scan (que lee secuencialmente pero puede estar en cache).

```ini
effective_cache_size = 10GB
```

Se recomienda entre el 50 % y el 75 % de la RAM total. Un valor demasiado bajo inclina al optimizador hacia Seq Scan innecesarios.

## maintenance_work_mem

Es la memoria disponible para operaciones de mantenimiento: `VACUUM`, `CREATE INDEX`, `REINDEX`, `ALTER TABLE ADD FOREIGN KEY`. Al ser operaciones que se ejecunan una a la vez (tipicamente), se puede asignar mas memoria que con `work_mem`.

```ini
maintenance_work_mem = 1GB
```

Un valor adecuado acelera significativamente la creacion de indices y la limpieza con VACUUM. El limite practico es la RAM disponible menos lo que consumen las conexiones activas.

## random_page_cost

Representa el costo relativo de una lectura de pagina aleatoria frente a una secuencial. El valor por defecto es 4.0, que refleja la relacion de rendimiento de discos magneticos tradicionales (HDD). En almacenamiento SSD, el costo de acceso aleatorio es mucho menor, por lo que se recomienda reducirlo a 1.0-1.5.

```ini
# Para SSD
random_page_cost = 1.1
```

Un `random_page_cost` sobredimensionado hace que el optimizador subestime los Index Scan y prefiera Seq Scan aunque no sea optimo.

## effective_io_concurrency

Controla cuantas operaciones de E/S concurrentes puede realizar PostgreSQL. En HDD tiene poco efecto (una unica cabeza lectora), pero en SSD o almacenamiento con multiples discos permite solapar lecturas.

```ini
# Para SSD
effective_io_concurrency = 200
```

El valor maximo recomendado esta entre 200 y 300 para SSD modernos. Este parametro afecta a los Bitmap Scan y a la precarga de datos en Index Scan.

## Resumen practico de ajuste

Para un servidor dedicado con 32 GB de RAM y SSD:

```ini
shared_buffers = 8GB
effective_cache_size = 20GB
work_mem = 32MB
maintenance_work_mem = 2GB
random_page_cost = 1.1
effective_io_concurrency = 200
```

Estos valores son un punto de partida. La afinacion real requiere observar el comportamiento con la carga de trabajo concreta, especialmente la presencia de archivos temporales (work_mem) y la tasa de aciertos en shared_buffers.

