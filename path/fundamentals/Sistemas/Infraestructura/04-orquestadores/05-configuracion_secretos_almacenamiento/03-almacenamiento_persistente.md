# Almacenamiento persistente en orquestadores

Los contenedores son intrinsecamente efimeros: su sistema de archivos desaparece cuando el contenedor se detiene o se reprograma en otro nodo. Esta caracteristica es beneficiosa para la escalabilidad y la gestion del ciclo de vida, pero plantea un desafio fundamental para cualquier aplicacion que necesite preservar datos entre reinicios, como bases de datos, colas de mensajes o sistemas de ficheros compartidos. Los orquestadores abordan este problema mediante un sistema de abstracciones de almacenamiento que separa la necesidad de almacenamiento (declarada por la aplicacion) de la implementacion concreta (proporcionada por la infraestructura). En Kubernetes, este sistema se compone de volumenes efimeros para datos temporales, Persistent Volumes para recursos de almacenamiento del cluster, Persistent Volume Claims para las peticiones de las aplicaciones, y Storage Classes para la definicion tipificada de tipos de almacenamiento con provisionado dinamico.

## Volumenes efimeros: emptyDir

Los volumenes efimeros representan la forma mas basica de almacenamiento compartido entre contenedores dentro de un mismo pod. Un volumen `emptyDir` se crea cuando el pod es asignado a un nodo y persiste mientras el pod esta activo en ese nodo. Si el contenedor se reinicia dentro del mismo pod, los datos se conservan; sin embargo, si el pod es eliminado o reprogramado a otro nodo, el contenido del volumen se pierde permanentemente. Este comportamiento lo hace ideal para datos temporales como caches, archivos de intercambio entre contenedores o datos de diagnostico.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-con-volumen-temporal
  namespace: produccion
spec:
  containers:
    - name: procesador
      image: mi-app-procesadora:1.0
      volumeMounts:
        - name: datos-temporales
          mountPath: /tmp/procesamiento
    
    - name: sidecar-logs
      image: fluentd:latest
      volumeMounts:
        - name: datos-temporales
          mountPath: /var/log/app
          readOnly: true
  
  volumes:
    - name: datos-temporales
      emptyDir:
        sizeLimit: 500Mi  # Limite de espacio para este volumen
        medium: ""  # "" usa disco normal, "Memory" usa tmpfs (RAM)
```

> El tipo `medium: "Memory"` almacena el volumen en RAM del nodo en lugar de disco. Es ideal para datos que requieren acceso ultra-rapido pero debe usarse con cuidado, ya que consume memoria que de otro modo estaria disponible para la cache del sistema operativo y otros procesos.

## Persistent Volumes y Persistent Volume Claims

Los Persistent Volumes (PV) representan recursos de almacenamiento de nivel cluster que existen de forma independiente del ciclo de vida de los pods. Un PV es un fragmento de almacenamiento provisionado por un administrador del cluster o dinamicamente por un Storage Class, que se ha configurado previamente en el cluster. Los Persistent Volume Claims (PVC) son las peticiones de almacenamiento que realizan los usuarios o aplicaciones: un PVC declara la cantidad de almacenamiento y las caracteristicas deseadas (tipo, modo de acceso, etc.), y el sistema de scheduling del cluster busca un PV que satisfaga esos requisitos y lo asigna al PVC. Esta separacion permite que las aplicaciones soliciten almacenamiento sin conocer los detalles de implementacion subyacente.

```yaml
# Persistent Volume - recurso de nivel cluster
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-postgresql-01
  labels:
    tipo: postgresql
    entorno: produccion
spec:
  capacity:
    storage: 50Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce  # Montable en un solo nodo en modo escritura
  persistentVolumeReclaimPolicy: Retain  # Conservar datos al liberar el PV
  storageClassName: ssd-rapido
  mountOptions:
    - noatime  # Optimizacion: no actualizar tiempos de acceso
    - nodiratime
  # Implementacion concreta (ejemplo: AWS EBS)
  awsElasticBlockStore:
    volumeID: vol-0abc123def456789
    fsType: ext4
  
---

# Persistent Volume Claim - peticion de almacenamiento
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-postgresql-datos
  namespace: produccion
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 30Gi
  storageClassName: ssd-rapido
  selector:
    matchLabels:
      tipo: postgresql  # Buscar PVs con esta etiqueta

---

# Pod usando el PVC
apiVersion: v1
kind: Pod
metadata:
  name: postgresql-pod
  namespace: produccion
spec:
  containers:
    - name: postgresql
      image: postgres:16
      ports:
        - containerPort: 5432
      env:
        - name: POSTGRES_DB
          value: app_production
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: DATABASE_PASSWORD
      volumeMounts:
        - name: datos-postgresql
          mountPath: /var/lib/postgresql/data
          subPath: pgdata  # Subdirectorio dentro del volumen
  
  volumes:
    - name: datos-postgresql
      persistentVolumeClaim:
        claimName: pvc-postgresql-datos
```

> Los modos de acceso `ReadWriteOnce` (RWO) permiten montaje en un solo nodo, `ReadOnlyMany` (ROX) permite lectura desde multiples nodos, y `ReadWriteMany` (RWX) permite escritura concurrente desde multiples nodos. No todos los tipos de almacenamiento soportan todos los modos.

## Storage Classes y provisionado dinamico

Los Storage Classes eliminan la necesidad de provisionar Persistent Volumes manualmente. Definen "perfiles" de almacenamiento que incluyen el proveedor subyacente, las caracteristicas de rendimiento y la politica de reclamacion. Cuando un PVC hace referencia a un Storage Class, el orquestador invoca automaticamente al provisionador asociado para crear el PV necesario en la infraestructura subyacente, ya sea un disco EBS en AWS, un Persistent Disk en GCP, o un LUN en un array SAN local.

```yaml
# Storage Class para almacenamiento SSD rapido
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ssd-rapido
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: ebs.csi.aws.com  # CSI driver para AWS EBS
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
  fsType: ext4
reclaimPolicy: Retain
allowVolumeExpansion: true  # Permitir redimensionar PVCs
volumeBindingMode: WaitForFirstConsumer  # Asignar PV cuando el pod sea scheduling
mountOptions:
  - noatime
  - discard  # Habilitar TRIM para SSDs

---

# Storage Class para almacenamiento HDD economico
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: hdd-economico
provisioner: ebs.csi.aws.com
parameters:
  type: st1
  encrypted: "true"
  fsType: xfs
reclaimPolicy: Delete  # Eliminar PV al liberar el PVC
allowVolumeExpansion: true

---

# Storage Class para almacenamiento compartido NFS
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-compartido
provisioner: nfs.csi.k8s.io
parameters:
  server: nfs-server.cluster.local
  share: /exports/compartido
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: Immediate
```

> `volumeBindingMode: WaitForFirstConsumer` retrasa la asignacion del PV hasta que un pod que usa el PVC es scheduling. Esto es critico en clusters multi-zona, ya que asegura que el PV se cree en la misma zona que el nodo donde se ejecutara el pod, evitando problemas de latencia por acceso a traves de la red.

## Volumenes en StatefulSets

Los StatefulSets son el recurso de orquestador disenado especificamente para aplicaciones con estado, como bases de datos distribuidas, clusters de caché o sistemas de cola. A diferencia de un Deployment, los StatefulSets garantizan que cada pod tiene un identificador unico y estable, y proporcionan almacenamiento persistente individualizado por cada réplica. Cada pod en un StatefulSet recibe su propio PVC automaticamente, lo que permite que cada instancia de la base de datos mantenga sus datos de forma independiente.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql-cluster
  namespace: produccion
spec:
  serviceName: postgresql-headless
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
        - name: postgresql
          image: postgres:16
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: datos-postgresql
              mountPath: /var/lib/postgresql/data
              subPath: pgdata
          env:
            - name: POSTGRES_DB
              value: app_production
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: DATABASE_PASSWORD
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
  
  volumeClaimTemplates:
    - metadata:
        name: datos-postgresql
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: ssd-rapido
        resources:
          requests:
            storage: 100Gi
```

> Los `volumeClaimTemplates` crean automaticamente un PVC para cada replica del StatefulSet, con un nombre derivado del patron `{nombre-template}-{nombre-statefulset}-{ordinal}`. Por ejemplo: `datos-postgresql-postgresql-cluster-0`.

