# Gestion de Nodos

La disponibilidad y salud de los nodos worker constituye la base sobre la cual se sostiene toda la infraestructura de un cluster Kubernetes. Un nodo que falla sin ser detectado a tiempo puede dejar pods sin recursos, provocar interrupciones en servicios criticos y generar cascadas de fallos en dependencias. La gestion de nodos abarca desde la deteccion de fallos hasta el escalado automatico de la capacidad del cluster, pasando por operaciones de mantencion planificada que requieren la evacuacion controlada de las cargas de trabajo. Comprender estos mecanismos es esencial para operar clusters de produccion con los niveles de disponibilidad exigidos por organizaciones modernas.

El modelo de gestion de nodos en Kubernetes se fundamenta en la separacion entre el **control plane** (que toma decisiones de orquestacion) y los **nodos worker** (que ejecutan las cargas de trabajo). Esta arquitectura distribuida permite que el control plane siga operando incluso cuando varios nodos worker fallan simultáneamente, siempre que exista suficiente capacidad de reprogramacion en los nodos restantes.

## Deteccion de Fallos por Heartbeat

Kubernetes implementa un mecanismo de deteccion de fallos basado en **heartbeats** periodicos que los nodos envian al control plane. El kubelet en cada nodo actualiza periodicamente el objeto `Node` en el API server, informando sobre sus condiciones: `Ready`, `MemoryPressure`, `DiskPressure`, `PIDPressure` y `NetworkUnavailable`. Cuando el API server deja de recibir actualizaciones de un nodo durante un periodo configurado (`--node-monitor-period`, por defecto 5 segundos, y `--node-monitor-grace-period`, por defecto 40 segundos), marca el nodo como `NotReady` y activa el proceso de eviction de pods.

```bash
# verificacion del estado de los nodos
kubectl get nodes -o wide
kubectl get nodes -o custom-columns=NAME:.metadata.name,STATUS:.status.conditions[-1].type,HEARTBEAT:.status.conditions[-1].lastHeartbeatTime

# inspeccion de condiciones de un nodo
kubectl describe node worker-03 | grep -A10 "Conditions:"

# verificacion de eventos de nodos
kubectl get events --field-selector reason=NodeNotReady --sort-by='.lastTimestamp'

# monitoreo continuo del estado de nodos
kubectl get nodes --watch -o custom-columns=NAME:.metadata.name,STATUS:.status.conditions[-1].type,REASON:.status.conditions[-1].reason
```

> El `node-monitor-grace-period` define cuanto tiempo espera el control plane antes de considerar que un nodo ha fallado. En entornos con latencia de red elevada o nodos con alta carga, este valor puede incrementarse a 60-120 segundos para evitar falsos positivos. Sin embargo, incrementarlo demasiado retrasa la deteccion de fallos reales.

## Cordon y Drain de Nodos

Las operaciones de mantencion de nodos — actualizaciones del sistema operativo, reemplazo de hardware, parches de seguridad — requieren la evacuacion controlada de las cargas de trabajo antes de apagar o reiniciar el nodo. El proceso se divide en dos fases: **cordon**, que marca el nodo como no apto para recibir nuevos pods, y **drain**, que migra los pods existentes a otros nodos del cluster.

El comando `kubectl cordon` establece la etiqueta `node.kubernetes.io/unschedulable=true` en el nodo, impidiendo que el scheduler ubique nuevos pods en el. Esto es una operacion reversable que no afecta a los pods ya en ejecución. El comando `kubectl drain` es mas agresivo: despues de cordoning el nodo, procede a evictar los pods existentes, respetando los PodDisruptionBudgets y las anotaciones de `do-not-drain` configuradas.

```bash
# fase 1: marcar el nodo como no scheduleable
kubectl cordon worker-03

# fase 2: evacuar los pods del nodo
kubectl drain worker-03 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60 \
  --timeout=300s

# verificacion de que el nodo esta vacio
kubectl get pods -o wide --field-selector spec.nodeName=worker-03

# fase 3: ejecutar la mantencion (reinicio, actualizacion, etc.)
# ... operaciones de sistema ...

# fase 4: habilitar el nodo nuevamente
kubectl uncordon worker-03
```

> El flag `--ignore-daemonsets` es esencial porque los DaemonSets estan diseñados para ejecutarse en todos los nodos (o en un subconjunto filtrado por nodSelector). Intentar evictar DaemonSets provocaria un bucle donde el controlador recrea el pod inmediatamente despues de ser evictado. Del mismo modo, `--delete-emptydir-data` es necesario para pods con volumenes emptyDir que contienen datos temporales que no necesitan persistirse.

## Escalado Automatico de Nodos con Cluster Autoscaler

El **Cluster Autoscaler** extiende el concepto de autoscaling al nivel de infraestructura, añadiendo o eliminando nodos worker automaticamente segun la demanda de los pods. Cuando el scheduler detecta que existen pods con `Pending` status debido a insuficiente capacidad de CPU o memoria en los nodos existentes, el Cluster Autoscaler solicita la creacion de nuevos nodos al proveedor de nube. Inversamente, cuando los nodos quedan subutilizados — todos sus pods pueden reubicarse en otros nodos — el autoscaler reduce la capacidad para optimizar costos.

La interaccion entre el scheduler y el Cluster Autoscaler se basa en la capacidad de **preemption** y **bin packing**: el scheduler intenta colocar los pods en los nodos disponibles utilizando estrategias de compactacion, y solo cuando no existe capacidad suficiente en ningun nodo, el Cluster Autoscaler evalua si es necesario añadir capacidad.

```yaml
# configuracion basica del Cluster Autoscaler en GKE
# (la configuracion especifica varia segun el proveedor)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
      - name: cluster-autoscaler
        image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.28.0
        command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/my-cluster
        - --balance-similar-node-groups
        - --skip-nodes-with-system-pods=false
```

> El `expander` define la estrategia para seleccionar que grupo de nodos crear cuando multiples opciones estan disponibles. `least-waste` selecciona el grupo que deja menor porcentaje de recursos desaprovechados, `random` selecciona al azar, y `priority` utiliza un ranking definido por el operador. En clusters con tipos de instancia heterogeneos, `least-waste` es la opcion mas eficiente en terminos de costos.

## Estrategias de Expancion y Contraccion

El Cluster Autoscaler implementa algoritmos sofisticados para determinar cuando y como ajustar la capacidad del cluster. La **expancion** se activa cuando existen pods en estado `Pending` que no pueden ser programados en ningun nodo disponible. La **contraccion** se basa en la evaluacion de la utilizacion de cada nodo: si todos los pods de un nodo pueden ser reubicados en otros nodos sin exceder sus capacidades, el nodo es candidato a ser eliminado.

```bash
# configuracion de grupos de nodos con limites
# en AWS EKS con eksctl
eksctl create nodegroup \
  --cluster=my-cluster \
  --name=workers \
  --node-type=t3.medium \
  --nodes=2 \
  --nodes-min=1 \
  --nodes-max=10 \
  --node-labels=workload=general \
  --node-taints="dedicated=general:NoSchedule"

# verificacion de la configuracion del autoscaler
kubectl get configmap cluster-autoscaler-status -n kube-system -o yaml

# monitoreo de eventos de escalado
kubectl get events -n kube-system --field-selector reason=ScalingUp --sort-by='.lastTimestamp'
kubectl get events -n kube-system --field-selector reason=ScalingDown --sort-by='.lastTimestamp'

# consulta de nodos disponibles y su utilizacion
kubectl top nodes
kubectl describe nodes | grep -E "Name:|Allocated resources:"
```

> El `scale-down-delay-after-add` (por defecto 10 minutos) y `scale-down-unneeded-time` (por defecto 10 minutos) son criticos para evitar flapping en la capacidad del cluster. Estos valores garantizan que un nodo no sea eliminado inmediatamente despues de ser añadido o de que sus pods terminen, permitiendo que la carga se estabilice antes de tomar decisiones de contraccion.

