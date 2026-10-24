# Restauracion de Backups en MongoDB

La restauracion es la unica prueba real de que un backup es valido. Un archivo BSON o un snapshot que nunca se ha restaurado no es un backup, es un talisman. MongoDB expone distintos caminos de restauracion segun el metodo de backup empleado, el nivel de consistencia requerido y la ventana de tiempo disponible para la recuperacion.

## Restauracion con mongorestore y oplogReplay

Cuando se dispone de un dump generado con `--oplog`, la restauracion debe aplicar las operaciones registradas durante la ventana de captura para alcanzar un estado consistente. El procedimiento estandar:

```bash
mongorestore --uri="mongodb://localhost:27017" \
  --oplogReplay \
  --dir=/backups/with_oplog
```

La opcion `--oplogReplay` reproduce secuencialmente las operaciones contenidas en `oplog.bson`. MongoDB garantiza que, tras la reproduccion, todas las colecciones reflejan el mismo punto logico en el tiempo, eliminando cualquier inconsistencia entre documentos y colecciones relacionadas que pudiera haber surgido por escrituras concurrentes durante el dump.

Si el dump no incluye `--oplog`, la restauracion sigue siendo funcional pero el estado resultante es el de "mejor esfuerzo": cada coleccion refleja el momento exacto en que mongodump comenzo a leerla, no un instante comun.

Para restauraciones selectivas se usan los filtros de namespace:

```bash
mongorestore --nsInclude="midb.*" --nsExclude="midb.logs" /backups/full
```

Esto permite recuperar solo las bases de datos o colecciones necesarias, agilizando el proceso cuando el fallo es parcial.

## Point-in-time recovery (PITR)

La recuperacion a un punto exacto en el tiempo requiere dos componentes: un snapshot base (completo) y el oplog completo que cubre desde ese snapshot hasta el instante objetivo. El proceso:

1. Restaurar el snapshot base con mongorestore:
   ```bash
   mongorestore --drop /backups/20260726_full
   ```
2. Reproducir el oplog desde el timestamp del snapshot hasta el momento deseado:
   ```bash
   mongorestore --oplogReplay --oplogLimit="1728000000:<timestamp_objetivo>" /backups/oplog_archive
   ```

El parametro `--oplogLimit` detiene la reproduccion en un timestamp especifico. El timestamp de MongoDB es un valor de 64 bits compuesto por los segundos desde la epoch Unix (32 bits) mas un contador ordinal (32 bits), accesible mediante `new Timestamp()` en la shell.

En Atlas, el PITR se configura desde la interfaz web seleccionando el cluster, la pestana "Backup" y la opcion "Restore". Atlas permite elegir un timestamp con granularidad de segundos dentro de la ventana de retencion del oplog (tipicamente 24 horas en planes M10+).

> El PITR requiere que el oplog este disponible de forma continua desde el snapshot base hasta el punto de recuperacion. Si el oplog se ha truncado (por ejemplo, al superar el tamano configurado de `oplogSizeMB`), la recuperacion solo llegara hasta el punto mas reciente disponible.

## Restore desde cloud: Atlas y S3

Atlas ofrece tres modalidades de restauracion:

- **Restore a un nuevo cluster**: Atlas crea un cluster temporal con los datos del snapshot seleccionado. Util cuando se necesita verificar el contenido antes de sobrescribir el cluster de produccion.
- **Restore in-place**: reemplaza los datos del cluster actual con los del snapshot. Implica downtime durante la operacion.
- **Download**: genera un enlace para descargar el snapshot en formato BSON, permitiendo restaurarlo en una instancia local o en otro proveedor.

```bash
# Iniciar restore a nuevo cluster via API
curl -s -u "apiKey:$(cat atlas_api_key)" -X POST \
  "https://cloud.mongodb.com/api/atlas/v1.0/groups/{GROUP_ID}/clusters/{CLUSTER_NAME}/restoreJobs" \
  -H "Content-Type: application/json" \
  -d '{
    "snapshotId": "{SNAPSHOT_ID}",
    "deliveryType": "download"
  }'
```

Para snapshots EBS almacenados en AWS, la restauracion consiste en crear un nuevo volumen desde el snapshot, montarlo en una instancia EC2 y arrancar MongoDB apuntando al directorio de datos restaurado:

```bash
aws ec2 create-volume --snapshot-id snap-xxxxxxx --availability-zone us-east-1a
# Montar en /mnt/data y arrancar mongod
mongod --dbpath /mnt/data
```

> Una restauracion desde snapshot EBS debe hacerse en una instancia con la misma version de MongoDB que la original. Las versiones de WiredTiger no siempre son compatibles hacia atras.

## Restore logico desde JSON/CSV

La restauracion desde formatos JSON o CSV (generados con mongoexport) se realiza con mongoimport. Este metodo no recrea indices ni preserva los tipos BSON complejos (ObjectId, Date, BinData) si no se respeta el formato `extended JSON`:

```bash
mongoimport --uri="mongodb://localhost:27017/midb" \
  --collection=usuarios \
  --file=usuarios.json \
  --mode=merge \
  --maintainInsertionOrder
```

El flag `--mode` acepta tres valores:
- `insert`: anyade documentos nuevos; falla si ya existe un `_id` duplicado.
- `upsert`: actualiza documentos existentes o inserta si no existen (requiere `--upsertFields`).
- `merge`: actualiza campos existentes y anyade los que faltan, sin sobrescribir el documento completo.

La restauracion logica desde JSON/CSV es adecuada para migraciones entre sistemas heterogeneos o para poblar entornos de desarrollo, pero no para recuperacion ante desastres: es lenta, no preserva la integridad referencial y no escala a conjuntos de datos grandes.

