# Estrategias de Backup y Recuperacion

Un backup tecnico sin una estrategia que defina frecuencias, retentiones y procedimientos de verificacion es simplemente un archivo durmiendo en disco. La estrategia convierte la copia de seguridad en un proceso gobernado por politicas medibles: RPO (Recovery Point Objective), RTO (Recovery Time Objective) y coste de almacenamiento. En MongoDB, estos parametros determinan que herramientas usar y como combinarlas.

## Backup completo mas incrementales

La combinacion de backups completos periodicos con capturas incrementales frecuentes equilibra coste de almacenamiento y velocidad de recuperacion. En MongoDB, el concepto de "incremental" se implementa mediante el oplog:

- **Backup completo semanal**: mongodump con `--oplog` o snapshot del sistema de archivos. Proporciona la base sobre la que se aplicaran los incrementales.
- **Backups incrementales diarios (o cada N horas)**: captura del oplog desde el ultimo backup completo. Se almacena como una secuencia de operaciones BSON.

```bash
# Backup completo (domingo)
mongodump --oplog --out=/backups/weekly/20260726

# Backup incremental del oplog (lunes)
mongosh --eval 'db.getSiblingDB("local").oplog.rs.find({ts: {$gt: Timestamp(1728000000, 1)}}).toArray()' \
  > /backups/incremental/20260727_oplog.json
```

Para restaurar desde un esquema completo+incremental:
1. Restaurar el ultimo backup completo: `mongorestore --oplogReplay /backups/weekly/20260726`
2. Aplicar los incrementales en orden cronologico: `mongorestore --oplogReplay /backups/incremental/20260727`

En la practica, los snapshots de EBS simplifican este esquema porque AWS gestiona los incrementales automaticamente: un snapshot EBS es incremental por naturaleza y solo consume almacenamiento por los bloques modificados desde el snapshot anterior, sin necesidad de scripts adicionales.

## Retencion de backups

La politica de retencion define cuantos backups se conservan y durante cuanto tiempo, en funcion de la criticidad de los datos y las restricciones de coste. Una politica de referencia:

| Frecuencia | Retencion | Proposito |
|---|---|---|
| Diaria | 7 dias | Restauracion rapida de errores recientes |
| Semanal | 4 semanas | Recuperacion de fallos detectados tarde |
| Mensual | 12 meses | Auditoria, cumplimiento legal, forensia |

La implementacion en un sistema Linux con cron y scripts bash:

```bash
#!/bin/bash
# /usr/local/bin/mongo_backup_rotation.sh

BACKUP_DIR="/backups"
DATE=$(date +%Y%m%d)

# Backup diario
mongodump --oplog --out="$BACKUP_DIR/daily/$DATE"

# Rotacion diaria: mantener 7 dias
find "$BACKUP_DIR/daily" -mindepth 1 -maxdepth 1 -mtime +7 -exec rm -rf {} \;

# Si es domingo, copiar como semanal
if [ "$(date +%u)" -eq 7 ]; then
  cp -al "$BACKUP_DIR/daily/$DATE" "$BACKUP_DIR/weekly/$DATE"
  find "$BACKUP_DIR/weekly" -mindepth 1 -maxdepth 1 -mtime +28 -exec rm -rf {} \;
fi

# Si es primer dia del mes, copiar como mensual
if [ "$(date +%d)" -eq 1 ]; then
  cp -al "$BACKUP_DIR/daily/$DATE" "$BACKUP_DIR/monthly/$DATE"
  find "$BACKUP_DIR/monthly" -mindepth 1 -maxdepth 1 -mtime +365 -exec rm -rf {} \;
fi
```

El uso de `cp -al` crea hard links, de modo que los backups semanales y mensuales no duplican espacio en disco mientras los archivos no cambien. En sistemas de archivos COW (Copy-on-Write) como ZFS o Btrfs, la rotacion se delega al propio sistema de snapshots del filesystem.

## Testing de restore

Un backup que no se prueba periodicamente no es fiable. El testing de restore debe ejecutarse de forma automatica y programada, verificando:

1. **Integridad del BSON**: mongorestore con `--dryRun` o validacion de checksums.
2. **Consistencia de datos**: conteo de documentos en colecciones criticas, verificacion de que los indices se reconstruyen correctamente.
3. **Rendimiento de restore**: el tiempo real de restauracion debe estar dentro del RTO acordado.

Un flujo tipico de testing automatizado:

```bash
# Entorno de staging
mongorestore --uri="mongodb://staging:27017" --drop /backups/daily/latest

# Validacion
mongosh --eval '
  const dbs = db.adminCommand("listDatabases");
  dbs.databases.forEach(d => {
    const colls = db.getSiblingDB(d.name).getCollectionNames();
    colls.forEach(c => {
      print(d.name + "." + c + ": " + db.getSiblingDB(d.name).getCollection(c).countDocuments());
    });
  });
'
```

En equipos maduros, el testing de restore forma parte del pipeline de CI/CD: tras cada backup completo, se dispara un job que lo restaura en un entorno efimero (Docker o Kubernetes), ejecuta validaciones y destruye el entorno al finalizar.

## Automatizacion: cron, Atlas y CI/CD

### cron (infraestructura on-premise)

El metodo clasico: scripts bash ejecutados por el demonio cron del sistema operativo. Ejemplo de entrada en `/etc/crontab`:

```
0 2 * * * root /usr/local/bin/mongo_backup_full.sh     # 02:00 diario
0 3 * * 0 root /usr/local/bin/mongo_backup_rotate.sh   # 03:00 domingo (rotacion)
*/30 * * * * root /usr/local/bin/mongo_backup_oplog.sh  # cada 30 min (oplog)
```

Las limitaciones de cron saltan a la vista: no hay notificaciones de fallo, no hay reintentos, no hay metricas. Para entornos no criticos puede ser suficiente, pero en produccion es recomendable envolver los scripts en un orquestador como systemd timers con alertas integradas.

### Atlas (cloud gestionado)

En MongoDB Atlas, la automatizacion de backups es nativa. Se configura desde la consola de administracion sin necesidad de scripting:

```bash
# Habilitar backups via API
curl -s -u "apiKey:$(cat atlas_api_key)" -X PATCH \
  "https://cloud.mongodb.com/api/atlas/v1.0/groups/{GROUP_ID}/clusters/{CLUSTER_NAME}" \
  -H "Content-Type: application/json" \
  -d '{
    "backupEnabled": true,
    "pitEnabled": true,
    "retentionDays": 7,
    "weeklyRetentionPeriod": 4,
    "monthlyRetentionPeriod": 12
  }'
```

Atlas notifica automaticamente si un backup falla, permite descargar logs de backup y expone metricas de tamano y exito via el dashboard de monitoreo.

### CI/CD (infraestructura como codigo)

Para equipos que gestionan MongoDB con Terraform, Ansible o Kubernetes, los backups se definen como parte del codigo de infraestructura:

```yaml
# Ejemplo con Kubernetes CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mongodb-backup
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: mongodump
            image: mongo:7.0
            command:
            - mongodump
            - --uri=$(MONGODB_URI)
            - --oplog
            - --out=/backups/$(date +%Y%m%d)
          restartPolicy: OnFailure
```

Este enfoque garantiza que la configuracion de backups esta versionada, revisable y desplegable junto con el resto de la infraestructura.

> Sin importar la herramienta de automatizacion elegida, la regla de oro del backup es: "restaura regularmente o pierde tus datos". Automatizar el backup es el primer paso; automatizar el test de restore es el que realmente protege los datos.

