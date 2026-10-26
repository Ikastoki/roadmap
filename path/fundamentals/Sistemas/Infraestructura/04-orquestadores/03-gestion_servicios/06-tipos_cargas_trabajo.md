# Tipos de cargas de trabajo

No todos los servicios que se ejecutan en un orquestador de contenedores tienen las mismas necesidades ni responden a los mismos patrones de ejecucion. Un servidor web que atiende miles de peticiones simultaneas tiene requisitos radicalmente distintos a una base de datos que necesita almacenamiento persistente y una identidad estable, o a un agente de monitoreo que debe ejecutarse en cada nodo del cluster. Los orquestadores como Kubernetes proporcionan diferentes tipos de recursos (workload resources) diseñados especificamente para cada patron, y elegir el tipo correcto es una de las decisiones arquitectonicas mas importantes al diseñar servicios en un entorno orquestado.

## Stateless (servicios sin estado)

Los servicios stateless son aquellos que no almacenan datos entre peticiones y no dependen de una identidad particular. Cada instancia es intercambiable con cualquier otra, lo que los hace ideales para escalado horizontal: si se necesita mas capacidad, simplemente se añaden mas replicas. Los servidores web, las APIs REST, los balances de carga y los proxies inversos son ejemplos tipicos. La ausencia de estado local significa que un cliente puede ser atendido por cualquier instancia, lo que simplifica la distribucion de trafico y la gestion de fallos: si una instancia muere, el balanceador de carga redirige las conexiones siguientes a otra instancia sin perdida de datos.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-frontend
  labels:
    app: web-frontend
    tier: frontend
spec:
  replicas: 4
  selector:
    matchLabels:
      app: web-frontend
  template:
    metadata:
      labels:
        app: web-frontend
        tier: frontend
    spec:
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "250m"
              memory: "256Mi"
```

## Stateful (servicios con estado)

Los servicios stateful requieren identidad estable, almacenamiento persistente y orden de arranque y apagado determinista. Una base de datos replicada, por ejemplo, necesita que cada nodo tenga un nombre predecible (para que los replicas sepan a que peer conectarse), que los datos sobrevivan reinicios de Pods, y que el ordende inicializacion sea conocido (el nodo primario debe estar listo antes que los secundarios). El StatefulSet en Kubernetes proporciona estas garantias: cada Pod recibe un indice ordinal estable (postgres-0, postgres-1, postgres-2), los Pods se crean y destruyen en orden estricto, y cada Pod tiene acceso a un PersistentVolumeClaim unico que se mantiene vinculado aunque el Pod se programe en un nodo diferente.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  labels:
    app: postgresql
spec:
  serviceName: postgres-headless
  replicas: 3
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
              name: postgres
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
              subPath: postgres
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: ssd-fast
        resources:
          requests:
            storage: 50Gi
---
# Servicio headless para resolver IPs individuales de cada Pod
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None
  selector:
    app: postgresql
  ports:
    - port: 5432
      targetPort: 5432
```

> El servicio headless (`clusterIP: None`) es fundamental para StatefulSets. En lugar de retornar una unica IP virtual que balancea entre todas las replicas, resuelve directamente las IPs de cada Pod. Esto permite que cada replica descubra las direcciones de sus peers mediante DNS: `postgres-0.postgres-headless.namespace.svc.cluster.local`, `postgres-1.postgres-headless...`, etc.

## DaemonSet (un nodo, una instancia)

Los DaemonSet garantizan que exactamente un Pod se ejecute en cada nodo del cluster (o en un subconjunto seleccionado por nodeselector/tolerations). Cuando se añade un nodo nuevo al cluster, el DaemonSet programa automaticamente un Pod en el. Cuando un nodo es eliminado, el Pod correspondiente se elimina junto con el. Este patron es ideal para agentes de infraestructura que necesitan acceso directo al host: coletores de metricas del nodo (Prometheus Node Exporter), agentes de logs (Fluentd, Filebeat), agentes de red (Calico, Cilium), herramientas de seguridad (Falco) y gestores de almacenamiento (Rook, Longhorn).

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitorizacion
  labels:
    app: prometheus-node-exporter
spec:
  selector:
    matchLabels:
      app: prometheus-node-exporter
  template:
    metadata:
      labels:
        app: prometheus-node-exporter
    spec:
      hostNetwork: true
      hostPID: true
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          effect: NoSchedule
      containers:
        - name: node-exporter
          image: prom/node-exporter:v1.7.0
          ports:
            - containerPort: 9100
              hostPort: 9100
          volumeMounts:
            - name: proc
              mountPath: /host/proc
              readOnly: true
            - name: sys
              mountPath: /host/sys
              readOnly: true
      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: sys
          hostPath:
            path: /sys
```

> Los DaemonSet suelen necesitar toleraciones para ejecutarse en nodos del plano de control (control-plane), que por defecto tienen un taint `NoSchedule`. Sin la toleracion adecuada, el agente no se programaria en esos nodos, dejando un hueco en la monitorizacion o seguridad del cluster.

## Batch y Jobs (cargas de trabajo por lotes)

Las cargas batch son tareas que se ejecutan hasta completarse y luego se detienen. No sirven peticiones de usuarios ni necesitan alta disponibilidad continua. Migraciones de bases de datos, procesamiento de imagenes, entrenamientos de modelos, backups programados y tareas ETL (Extract, Transform, Load) son ejemplos tipicos. Kubernetes proporciona dos recursos para este patron: Job (para tareas que deben completarse al menos una vez) y CronJob (para tareas que deben ejecutarse segun una programacion temporal). Un Job crea uno o mas Pods y se encarga de reiniciarlos si fallan hasta que el numero especificado de completaciones exitosas se alcanza.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: migracion-schema-v3
  namespace: produccion
spec:
  backoffLimit: 3
  activeDeadlineSeconds: 600
  ttlSecondsAfterFinished: 3600
  template:
    metadata:
      labels:
        app: migracion-db
    spec:
      restartPolicy: OnFailure
      containers:
        - name: migrador
          image: registry.empresa.com/db-migrator:v3.2.0
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: url
            - name: MIGRATION_VERSION
              value: "20240715_003"
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "1"
              memory: "1Gi"
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-diario
  namespace: produccion
spec:
  schedule: "0 2 * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 7
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: registry.empresa.com/pg-backup:latest
              env:
                - name: PGHOST
                  value: "postgres-0.postgres-headless"
                - name: PGDATABASE
                  value: "produccion"
              volumeMounts:
                - name: backup-storage
                  mountPath: /backups
          volumes:
            - name: backup-storage
              persistentVolumeClaim:
                claimName: backup-pvc
```

> `backoffLimit` define el numero maximo de reintentos antes de marcar el Job como fallido. `activeDeadlineSeconds` establece un tiempo maximo total de ejecucion, incluyendo reintentos. `ttlSecondsAfterFinished` controla cuanto tiempo persiste el Job y sus Pods despues de completarse: util para evitar la acumulacion de Jobs antiguos en el namespace sin necesidad de limpieza manual.

> `concurrencyPolicy: Forbid` en CronJob evita que se lance una nueva ejecucion si la anterior no ha terminado. Esto es critico para tareas como backups o migraciones donde ejecuciones simultaneas podrian causar corrupcion de datos o conflicto por recursos compartidos.

