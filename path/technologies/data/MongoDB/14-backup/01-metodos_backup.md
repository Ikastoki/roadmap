# Metodos de Backup en MongoDB

Un backup en MongoDB no es un simple volcado de archivos: la naturaleza de su motor de almacenamiento (WiredTiger por defecto) y su arquitectura distribuida (replicasets, sharding) imponen condiciones especificas para garantizar que la copia sea consistente y utilizable. MongoDB ofrece un abanico de herramientas que cubren desde entornos de un solo nodo hasta clusters shardeados en produccion, cada una con implicaciones distintas en rendimiento, consistencia y velocidad de restauracion.

## mongodump: backup logico en BSON

`mongodump` extrae los datos de MongoDB en formato BSON (Binary JSON), un binario que preserva los tipos de datos nativos de MongoDB. Es la herramienta mas utilizada por su simplicidad y portabilidad: un backup generado con mongodump puede restaurarse en cualquier version compatible de MongoDB, independientemente del motor de almacenamiento o la plataforma subyacente.

```bash
mongodump --uri="mongodb://localhost:27017" --out=/backups/full_$(date +%Y%m%d)
```

Por defecto, mongodump realiza una lectura secundaria sobre la base de datos, lo que significa que puede ejecutarse contra un secundario de un replicaset sin afectar al primario. El parametro `--oplog` es critico para entornos con replicacion:

```bash
mongodump --uri="mongodb://localhost:27017" --oplog --out=/backups/with_oplog
```

La opcion `--oplog` captura las operaciones de escritura que ocurren durante la ejecucion del dump, registrandolas en un archivo `oplog.bson`. Esto permite que la restauracion posterior refleje un punto en el tiempo consistente, no el estado fragmentado que resultaria de combinar datos capturados en distintos momentos. Sin `--oplog`, un dump de una base de datos con escrituras concurrentes puede contener inconsistencias entre colecciones relacionadas.

Limitaciones de mongodump:
- Impacto en rendimiento: fuerza lecturas en disco que compiten con la carga de trabajo normal.
- No preserva indices secundarios como archivos; los recrea durante la restauracion.
- Para bases de datos muy grandes (cientos de GB o TB), el tiempo de dump y restore puede ser prohibitivo.
- No captura snapshots a nivel de archivo, lo que impide restauraciones instantaneas.

## mongorestore: carga de BSON a MongoDB

`mongorestore` es la contraparte de mongodump. Carga archivos BSON en una instancia de MongoDB, recreando colecciones, documentos e indices. Su uso basico:

```bash
mongorestore --uri="mongodb://localhost:27017" /backups/full_20260728
```

Cuando se utiliza con un dump que incluye `--oplog`, mongorestore debe aplicarse con `--oplogReplay` para reproducir las operaciones capturadas y alcanzar un estado consistente:

```bash
mongorestore --uri="mongodb://localhost:27017" --oplogReplay /backups/with_oplog
```

El flag `--drop` es util en restauraciones completas: elimina las colecciones existentes antes de importar los datos, evitando duplicados. `--nsInclude` y `--nsExclude` permiten restaurar solo subconjuntos de bases de datos o colecciones, lo que resulta practico para recuperaciones parciales.

## mongoexport y mongoimport: formatos legibles

`mongoexport` produce archivos en JSON (formato legible) o CSV (compatible con hojas de calculo y herramientas ETL). A diferencia de mongodump, no exporta en BSON, sino en texto plano:

```bash
mongoexport --uri="mongodb://localhost:27017/mibd" --collection=usuarios --out=usuarios.json
mongoexport --uri="mongodb://localhost:27017/mibd" --collection=ventas --type=csv --fields=_id,cliente,total --out=ventas.csv
```

`mongoimport` realiza la operacion inversa:

```bash
mongoimport --uri="mongodb://localhost:27017/mibd" --collection=usuarios --file=usuarios.json
mongoimport --uri="mongodb://localhost:27017/mibd" --collection=ventas --type=csv --headerline --file=ventas.csv
```

Estas herramientas no son sustitutas de mongodump para backups: no preservan indices, no soportan --oplog y son significativamente mas lentas al procesar documento por documento. Su utilidad real reside en migraciones entre sistemas, alimentacion de data lakes, o exportacion de subsets para analisis. Un backup general que use mongoexport en lugar de mongodump es una mala practica.

## File system snapshots: LVM y EBS

Los snapshots a nivel de sistema de archivos capturan el estado completo del directorio de datos de MongoDB en un instante. Requieren que el motor de almacenamiento sea consistente con el snapshot, lo que en WiredTiger se logra porque es un motor MVCC (Multi-Version Concurrency Control) que mantiene puntos de control consistentes.

```bash
# Con LVM en Linux
lvm lvcreate --size 10G --snapshot --name mongo_snap /dev/vg0/mongodb
mount /dev/vg0/mongo_snap /mnt/snap
cp -a /mnt/snap/* /backups/lvm_snapshot/
umount /mnt/snap
lvm lvremove /dev/vg0/mongo_snap
```

Para snapshots en caliente (con MongoDB corriendo), es necesario forzar un checkpoint de WiredTiger y bloquear escrituras:

```bash
use admin
db.fsyncLock()
# Tomar snapshot
db.fsyncUnlock()
```

En entornos cloud, los snapshots de EBS (Amazon Web Services) proporcionan la misma funcionalidad con la ventaja de ser incrementales y almacenarse en S3:

```bash
aws ec2 create-snapshot --volume-id vol-xxxxxxx --description "MongoDB data snapshot $(date +%Y%m%d)"
```

Ventajas de los snapshots:
- Rapidez: la captura es casi instantanea independientemente del tamaño de los datos.
- Restauracion inmediata: montar el snapshot y arrancar MongoDB.
- No degradan el rendimiento durante la captura (a diferencia de mongodump).

Desventajas: estan vinculados al sistema de archivos y al hardware subyacente; no son portables entre plataformas ni entre versiones de MongoDB.

## Atlas backups: continuos, point-in-time y cross-region

MongoDB Atlas, el servicio gestionado de MongoDB, ofrece un sistema de backups integrado que elimina la gestion manual:

- **Backups continuos**: capturan los cambios en tiempo real a traves del oplog, permitiendo restaurar a cualquier punto en una ventana de hasta 24 horas (dependiendo del plan).
- **Point-in-time recovery (PITR)**: combina snapshots periodicos con el oplog almacenado para reconstruir el estado exacto en un timestamp especifico. En Atlas, se configura desde la consola de administracion y no requiere intervencion manual.
- **Cross-region snapshots**: replican los backups a una region secundaria de AWS/GCP/Azure para proteger frente a desastres regionales. Atlas retiene los snapshots segun una politica configurable (tipicamente diarios durante 7 dias, semanales durante 4 semanas, mensuales durante 12 meses).

Los backups de Atlas se activan por cluster y pueden descargarse como archivos BSON para restaurarlos en una instancia local o en otro proveedor cloud:

```bash
# Descarga via API de Atlas
curl -s -u "apiKey:$(cat atlas_api_key)" \
  "https://cloud.mongodb.com/api/atlas/v1.0/groups/{GROUP_ID}/clusters/{CLUSTER_NAME}/snapshots/{SNAPSHOT_ID}/restore" \
  -o snapshot_download.tar.gz
```

> Los backups de Atlas no tienen coste de licencia adicional, pero el almacenamiento de snapshots consume recursos facturables segun el plan contratado.

