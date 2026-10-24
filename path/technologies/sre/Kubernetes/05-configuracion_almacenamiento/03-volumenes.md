# Volumenes

Los contenedores son efimeros por diseno: cuando un Pod termina o se elimina, todo su sistema de archivos desaparece con el. Este comportamiento es aceptable para aplicaciones stateless que no necesitan recordar nada entre ejecuciones, pero fracasa estrepitosamente cuando la aplicacion necesita persistir datos, compartir informacion entre contenedores del mismo Pod, o acceder a discos externos. Un volumen en Kubernetes es el mecanismo que abstrae el almacenamiento y lo desacopla del ciclo de vida del contenedor.

Kubernetes soporta decenas de tipos de volumen, desde los mas sencillos (emptyDir, hostPath) hasta los que interactuan con la infraestructura de almacenamiento del proveedor cloud (EBS, Azure Disk, GCE Persistent Disk, NFS). La eleccion del tipo de volumen determina donde se almacenan los datos, como se gestiona su ciclo de vida y que garantias de persistencia ofrece.

## emptyDir

El volumen `emptyDir` es el mas simple. Se crea cuando un Pod se asigna a un nodo y existe mientras el Pod permanece en ese nodo. Cuando el Pod se elimina del nodo, el contenido del `emptyDir` se borra permanentemente. No es persistente, pero es util para compartir datos entre contenedores del mismo Pod o como almacenamiento temporal para operaciones de corta duracion.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-pod
spec:
  containers:
    - name: app
      image: mi-app:1.0
      volumeMounts:
        - name: cache
          mountPath: /var/cache
    - name: log-collector
      image: fluentd:latest
      volumeMounts:
        - name: cache
          mountPath: /var/logs
  volumes:
    - name: cache
      emptyDir:
        sizeLimit: 500Mi
```

Dos casos de uso tipicos para `emptyDir`:

- **Cache en disco**: una aplicacion escribe datos temporales que no necesitan persistir entre reinicios del Pod, pero cuyo acceso en disco es mas rapido que en red.
- **Sidecar de logs**: el contenedor principal escribe logs en un archivo dentro del volumen compartido, y un sidecar (Fluentd, Filebeat) los lee y los envia a un agregador central.

El `emptyDir` utiliza por defecto el disco local del nodo (el mismo donde se almacenan las imagenes y los contenedores). Es posible configurarlo para usar RAM en lugar de disco estableciendo `emptyDir.medium: Memory`, lo que proporciona alta velocidad pero consume memoria del nodo.

> Un `emptyDir` con `medium: Memory` cuenta contra el limite de memoria del Pod. Si el Pod excede el `sizeLimit` configurado, el kubelet puede expulsar el Pod del nodo.

## hostPath

`hostPath` monta un archivo o directorio del sistema de archivos del nodo dentro del Pod. A diferencia de `emptyDir`, los datos persisten incluso despues de que el Pod se elimine, porque residen en el nodo. Sin embargo, si el Pod se reprograma en otro nodo, los datos no lo siguen.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nodo-monitor
spec:
  containers:
    - name: monitor
      image: prometheus/node-exporter
      volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
  volumes:
    - name: proc
      hostPath:
        path: /proc
        type: Directory
```

`hostPath` no se recomienda en entornos productivos por varias razones. Rompe el aislamiento entre el Pod y el nodo: un contenedor puede acceder a archivos sensibles del sistema operativo del nodo. Ademas, el comportamiento no es portable: un Pod que funciona en un nodo Linux puede no funcionar en otro con distinta distribucion o estructura de directorios.

Los casos de uso aceptables para `hostPath` son contenedores que necesitan acceder al sistema de archivos del nodo por razones operativas: recolectores de metricas (Node Exporter), agentes de monitorizacion (Datadog agent), o herramientas de almacenamiento que gestionan directamente discos del nodo. En cualquier otro escenario, es preferible usar PersistentVolumeClaim.

## PersistentVolume y PersistentVolumeClaim

El sistema de PersistentVolume (PV) y PersistentVolumeClaim (PVC) abstrae la gestion del almacenamiento separando dos roles: el administrador del cluster provisiona los recursos de almacenamiento (PVs), y el desarrollador solicita almacenamiento para sus aplicaciones (PVCs). Kubernetes se encarga de emparejar las solicitudes con los recursos disponibles.

Un **PersistentVolume** es un recurso del cluster que representa un trozo de almacenamiento en la infraestructura subyacente. Puede ser un disco EBS en AWS, un disco persistente en GCP, un Azure Disk, un volumen NFS exportado por un servidor, o un directorio local en el nodo.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-ebs-001
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: fast-ssd
  awsElasticBlockStore:
    volumeID: vol-0abcdef1234567890
    fsType: ext4
```

Una **PersistentVolumeClaim** es una solicitud de almacenamiento por parte de un usuario. Especifica la capacidad minima, los modos de acceso y la clase de almacenamiento deseada. Kubernetes busca un PV que cumpla con los requisitos de la PVC y los vincula.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: fast-ssd
```

Cuando un Pod referencia la PVC, Kubernetes monta el PV vinculado en el directorio especificado:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
    - name: app
      image: mi-app:1.0
      volumeMounts:
        - name: data
          mountPath: /var/lib/data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: data-pvc
```

### Modos de acceso

Cada PV tiene un modo de acceso que determina cuantos nodos pueden montar el volumen simultaneamente y en que modo:

- **ReadWriteOnce (RWO)**: un solo nodo puede montar el volumen en modo lectura-escritura. Es el modo mas comun para discos en bloque (EBS, Azure Disk, GCE PD).
- **ReadOnlyMany (ROX)**: multiples nodos pueden montar el volumen en modo solo lectura.
- **ReadWriteMany (RWX)**: multiples nodos pueden montar el volumen en modo lectura-escritura. Solo algunos tipos de volumen lo soportan (NFS, Azure File, EFS).

> No todos los proveedores de almacenamiento soportan todos los modos de acceso. EBS solo soporta RWO. EFS y Azure File soportan RWX. Verificar el modo soportado es esencial al disenar la arquitectura de almacenamiento.

### Politicas de reclamacion

Cuando se elimina la PVC, el PV queda liberado. La politica `persistentVolumeReclaimPolicy` determina que ocurre con el PV:

- **Retain**: el PV se queda con sus datos. El administrador debe limpiarlo manualmente antes de reutilizarlo.
- **Delete**: el PV y el recurso de almacenamiento subyacente se eliminan automaticamente. Es el comportamiento por defecto en proveedores cloud.
- **Recycle**: se borra el contenido del PV (deprecado en versiones recientes).

## StorageClasses

StorageClass permite definir clases de almacenamiento con caracteristicas distintas. En lugar de crear PVs manualmente, el administrador define una o varias StorageClasses y los PVs se crean automaticamente bajo demanda cuando un usuario crea una PVC. Este mecanismo se conoce como provisionamiento dinamico.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  fsType: ext4
```

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard-hdd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: st1
  fsType: ext4
```

Cuando un usuario crea una PVC con `storageClassName: fast-ssd`, el provisionador de AWS EBS crea un volumen gp3 de 3000 IOPS, lo asocia al cluster, y Kubernetes crea automaticamente el PV correspondiente.

Cada proveedor cloud tiene su propio provisionador y sus propios parametros. Los nombres de los provisionadores siguen la convencion `proveedor.io/recurso`:

- `kubernetes.io/aws-ebs` o `ebs.csi.aws.com` (EBS)
- `kubernetes.io/azure-disk` o `disk.csi.azure.com` (Azure Disk)
- `pd.csi.storage.gke.io` (GCE Persistent Disk)
- `nfs.csi.k8s.io` (NFS)
- `hostpath.csi.k8s.io` (HostPath local)

La diferencia entre los provisionadores `kubernetes.io/*` (in-tree) y `*.csi.*` (CSI) es relevante. Los in-tree estan compilados dentro del codigo de Kubernetes y se consideran legacy. CSI (Container Storage Interface) es el estandar actual: los controladores se ejecutan como Pods independientes, son mantenidos por los propios proveedores y se actualizan independientemente de Kubernetes.

```yaml
# StorageClass para NFS con CSI
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-share
provisioner: nfs.csi.k8s.io
parameters:
  server: nfs-server.internal
  share: /exported/path
mountOptions:
  - hard
  - nfsvers=4.1
```

```yaml
# StorageClass para Azure Disk
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-premium-ssd
provisioner: disk.csi.azure.com
parameters:
  skuname: Premium_LRS
  cachingMode: ReadOnly
  fsType: ext4
```

## Eleccion del tipo de almacenamiento

La decision entre los diferentes tipos de volumen depende del caso de uso concreto. `emptyDir` es adecuado para datos temporales y cache intra-Pod. `hostPath` queda restringido a herramientas de sistema que operan sobre el nodo. PV/PVC con StorageClasses es la opcion correcta para cualquier aplicacion que necesite persistencia.

Para aplicaciones que requieren baja latencia y alto rendimiento (bases de datos relacionales, colas de mensajes), los discos en bloque premium (EBS gp3/io2, Azure Premium SSD, GCE PD-SSD) son la opcion natural, pero solo soportan RWO. Para aplicaciones que necesitan acceso concurrente desde multiples Pods (sistemas de archivos compartidos, almacenamiento de objetos), las opciones son NFS, EFS (AWS), Azure File, o soluciones como Ceph y GlusterFS desplegadas sobre Kubernetes.

```bash
# Ver recursos de almacenamiento
kubectl get pv
kubectl get pvc
kubectl get storageclass

# Ver detalles de un PVC vinculado
kubectl describe pvc data-pvc

# Borrar una PVC y su PV asociado (si reclaimPolicy es Delete)
kubectl delete pvc data-pvc
```

> El sistema PV/PVC no es gratuito en terminos operativos. Cada volumen en un proveedor cloud tiene un costo mensual, independientemente de si se usa o no. Es recomendable monitorizar los PVs no vinculados y establecer politicas de limpieza automatica para evitar costes innecesarios.

