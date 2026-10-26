# Buenas practicas de almacenamiento en orquestadores

El almacenamiento en entornos containerizados presenta una tension inherente entre la naturaleza efimera de los contenedores y la necesidad de preservar datos criticos de negocio. Las bases de datos transaccionales, las colas de mensajes, los sistemas de ficheros compartidos y los repositorios de artefactos requieren persistencia garantizada, redundancia y rendimiento predecible. Sin embargo, el mal diseno de la capa de almacenamiento es una de las fuentes mas frecuentes de perdida de datos y de interrupciones de servicio en clústeres de produccion. Este documento establece las directrices fundamentales para construir una estrategia de almacenamiento robusta, que incluye la separacion rigurosa de codigo y datos, la seleccion de soluciones de almacenamiento distribuido para alta disponibilidad, la implementacion de politicas de backup automatizadas y el dimensionamiento adecuado de los recursos de almacenamiento segun las caracteristicas de cada tipo de carga de trabajo.

## Separacion de codigo y datos

El principio fundamental del almacenamiento containerizado es que las imagenes deben contener exclusivamente codigo y dependencias de ejecucion, mientras que todos los datos generados durante la operacion del servicio deben residir en volumenes externos. Esta separacion permite actualizar, reemplazar o eliminar contenedores sin afectar a los datos de negocio, y es la base para lograr escalabilidad, recuperacion ante desastres y despliegues continuos sin tiempo de inactividad.

```yaml
# Patron correcto: aplicacion con volumenes para datos
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-produccion
  namespace: produccion
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mi-app
  template:
    metadata:
      labels:
        app: mi-app
    spec:
      containers:
        - name: aplicacion
          image: mi-app:2.1.0
          volumeMounts:
            # Datos de la aplicacion (base de datos local, logs, cache)
            - name: datos-app
              mountPath: /app/data
            # Configuracion externa
            - name: config-volumen
              mountPath: /app/config
              readOnly: true
      
      volumes:
        - name: datos-app
          persistentVolumeClaim:
            claimName: pvc-app-datos
        - name: config-volumen
          configMap:
            name: app-config

---
# Para bases de datos: SIEMPRE usar StatefulSet con PVC dedicado
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql
  namespace: produccion
spec:
  serviceName: postgresql
  replicas: 2
  selector:
    matchLabels:
      app: postgresql
  template:
    spec:
      containers:
        - name: postgresql
          image: postgres:16
          volumeMounts:
            - name: pgdata
              mountPath: /var/lib/postgresql/data
              subPath: pgdata
  volumeClaimTemplates:
    - metadata:
        name: pgdata
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: ssd-rapido
        resources:
          requests:
            storage: 200Gi
```

> Una imagen Docker que almacena datos en su capa de escritura (sin volumenes) pierde todos esos datos al ser reemplazada. En entornos de produccion, esto equivale a una perdida de datos total cada vez que se realiza un despliegue.

## Estrategias de backup automatizado

La retencion de datos sin una estrategia de backup verificada es una falsa sensacion de seguridad. Los backups deben ser automatizados, verificados periodicamente y almacenados en una ubicacion geograficamente separada del cluster principal. En Kubernetes, los backups se implementan mediante CronJobs que ejecutan herramientas especificas para cada tipo de almacenamiento, combinados con politicas de retencion que equilibran el uso de almacenamiento con los requisitos de recuperacion.

```yaml
# CronJob para backup automatico de bases de datos PostgreSQL
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-postgresql
  namespace: produccion
spec:
  schedule: "0 2 * * *"  # Diariamente a las 2:00 AM
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 7
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 3600
      template:
        spec:
          serviceAccountName: backup-service-account
          containers:
            - name: backup
              image: postgres:16
              command:
                - /bin/bash
                - -c
                - |
                  set -euo pipefail
                  
                  TIMESTAMP=$(date +%Y%m%d_%H%M%S)
                  BACKUP_FILE="/backup/postgresql_${TIMESTAMP}.sql.gz"
                  
                  echo "Iniciando backup de PostgreSQL..."
                  pg_dump -h postgresql-svc.produccion.svc.cluster.local \
                          -U postgres \
                          -d app_production \
                          --format=custom \
                          --compress=6 \
                          > "${BACKUP_FILE}"
                  
                  # Verificar integridad del backup
                  pg_restore -l "${BACKUP_FILE}" > /dev/null 2>&1
                  if [ $? -eq 0 ]; then
                    echo "Backup verificado exitosamente: ${BACKUP_FILE}"
                    ls -lh "${BACKUP_FILE}"
                  else
                    echo "ERROR: Backup corrupto, eliminando..."
                    rm -f "${BACKUP_FILE}"
                    exit 1
                  fi
                  
                  # Subir a S3 (requiere aws-cli)
                  aws s3 cp "${BACKUP_FILE}" \
                    s3://mi-bucket-backups/produccion/postgresql/ \
                    --storage-class STANDARD_IA
                  
                  # Eliminar backups locales antiguos (>7 dias)
                  find /backup -name "*.sql.gz" -mtime +7 -delete
                  
                  echo "Backup completado y sincronizado."
              env:
                - name: PGPASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: postgresql-secrets
                      key: POSTGRES_PASSWORD
              volumeMounts:
                - name: backup-storage
                  mountPath: /backup
          
          volumes:
            - name: backup-storage
              persistentVolumeClaim:
                claimName: pvc-backups-temporales
          restartPolicy: OnFailure
```

```bash
# Script de verificacion de backups (ejecutar manualmente periodicamente)
#!/bin/bash
set -euo pipefail

BUCKET="s3://mi-bucket-backups/produccion"
FECHA_ULTIMO_BACKUP=$(aws s3 ls "${BUCKET}/postgresql/" --recursive | \
  sort | tail -n1 | awk '{print $4}')

echo "Ultimo backup encontrado: ${FECHA_ULTIMO_BACKUP}"

# Descargar y verificar
aws s3 cp "${BUCKET}/postgresql/${FECHA_ULTIMO_BACKUP}" /tmp/verify.sql.gz
pg_restore -l /tmp/verify.sql.gz > /dev/null 2>&1

if [ $? -eq 0 ]; then
  echo "Verificacion: OK"
  rm /tmp/verify.sql.gz
else
  echo "ERROR: Backup corrupto detectado"
  exit 1
fi
```

## Soluciones de almacenamiento distribuido

Para clustes con requisitos de alta disponibilidad, el almacenamiento local de un solo nodo es insuficiente. Las soluciones de almacenamiento distribuido replican los datos entre multiples nodos, proporcionando tolerancia a fallos y, en algunos casos, acceso concurrente desde varios nodos simultaneamente. Cada solucion presenta compromisos distintos entre rendimiento, complejidad operativa y coste.

| Solucion | Tipo | Replicacion | Escenario ideal |
|----------|------|-------------|-----------------|
| Ceph (RBD) | Bloque | Sincrona/Asincrona | Bases de datos, VMs |
| Ceph (CephFS) | Ficheros | Sincrona | Ficheros compartidos, CMS |
| GlusterFS | Ficheros | Replicacion por volumen | Datos compartidos, logs |
| NFS | Ficheros | No nativa (requiere RAID) | Desarrollo, datos legados |
| Longhorn | Bloque | Sincrona | K8s nativo, multi-nodo |

### Ceph como solucion de almacenamiento unificado

Ceph es la plataforma de almacenamiento distribuido mas utilizada en entornos Kubernetes de produccion, proporcionando tanto almacenamiento de tipo bloque (RBD) como de tipo fichero (CephFS) sobre la misma infraestructura. Su arquitectura basada en CRUSH permite escalar horizontalmente sin puntos unicos de fallo, y su integracion con Kubernetes a traves de Rook permite su despliegue y gestion directamente como un workload del cluster.

```yaml
# Configuracion de StorageClass para Ceph RBD
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph-cluster
  pool: ssd-pool
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: rook-ceph-csi
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-ceph-csi
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer

---

# Configuracion de StorageClass para CephFS (acceso compartido)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cephfs-compartido
provisioner: rook-cephfs.csi.ceph.com
parameters:
  clusterID: rook-ceph-cluster
  fsName: cephfs
  csi.storage.k8s.io/provisioner-secret-name: rook-ceph-csi
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-ceph-csi
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Retain
allowVolumeExpansion: true
```

```bash
# Verificar el estado del cluster Ceph
kubectl -n rook-ceph exec -it rook-ceph-tools-xxxxx -- ceph status
kubectl -n rook-ceph exec -it rook-ceph-tools-xxxxx -- ceph osd tree
kubectl -n rook-ceph exec -it rook-ceph-tools-xxxxx -- rados df

# Verificar que los PVCs se estan provisionando correctamente
kubectl get pvc -A | grep Bound
kubectl get pv | grep ceph
```

## Monitoreo y alertas de almacenamiento

El agotamiento del espacio de disco es una de las causas mas comunes de interrupcion de servicio en clustes Kubernetes. Un pod que intenta escribir en un volumen lleno puede entrar en un bucle de reinicios continuos, degradar su rendimiento significativamente o perder datos transaccionales. La monitorizacion proactiva del uso de almacenamiento es esencial para anticipar problemas antes de que afecten a la disponibilidad del servicio.

```yaml
# Regla de Prometheus para alertar cuando un PVC esta al 85% de uso
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: alertas-almacenamiento
  namespace: monitoring
spec:
  groups:
    - name: almacenamiento
      rules:
        - alert: PVCUsoCritico
          expr: |
            (kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes) > 0.85
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "PVC {{ $labels.persistentvolumeclaim }} al {{ $value | humanizePercentage }}"
            description: >
              El PVC {{ $labels.persistentvolumeclaim }} en el namespace 
              {{ $labels.namespace }} esta usando mas del 85% de su capacidad.
        
        - alert: PVCUsoCritico
          expr: |
            (kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes) > 0.95
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "PVC {{ $labels.persistentvolumeclaim }} al {{ $value | humanizePercentage }} - RIESGO DE PERDIDA DE DATOS"
        
        - alert: PVCSinProvisionar
          expr: |
            kube_persistentvolumeclaim_status_phase{phase="Pending"} > 0
          for: 15m
          labels:
            severity: critical
          annotations:
            summary: "PVC {{ $labels.persistentvolumeclaim }} en estado Pending"
```

```bash
# Comando util para verificar el uso de almacenamiento del cluster
kubectl get pvc -A -o custom-columns=\
'NAMESPACE:.metadata.namespace,\
NAME:.metadata.name,\
STATUS:.status.phase,\
STORAGE:.spec.resources.requests.storage,\
VOLUME:.spec.volumeName'

# Verificar el espacio disponible en los nodos
kubectl describe nodes | grep -A 5 "Allocated resources"
```

> El monitoreo debe complementarse con politicas de retencion de datos y limpieza automatica de logs antiguos. Herramientas como `logrotate` dentro de los contenedores o sidecars de limpieza pueden prevenir el crecimiento indefinido del consumo de almacenamiento.

