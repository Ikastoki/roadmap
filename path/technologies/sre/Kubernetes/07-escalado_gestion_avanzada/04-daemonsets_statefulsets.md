# DaemonSets y StatefulSets

Los Deployments son el recurso adecuado para aplicaciones stateless donde cualquier replica es indistinta. Pero no todos los servicios encajan en ese modelo. Kubernetes proporciona dos controladores adicionales para escenarios especificos: DaemonSet, para ejecutar un Pod por nodo, y StatefulSet, para aplicaciones con estado que requieren identidad y orden.

## DaemonSet

Un DaemonSet garantiza que todos los nodos del cluster (o un subconjunto seleccionado) ejecuten una copia de un Pod. Cuando un nodo se une al cluster, el DaemonSet crea automaticamente el Pod en el. Cuando un nodo se elimina, el Pod se recolecta.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  selector:
    matchLabels:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluent/fluentd
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

Los casos de uso tipicos del DaemonSet incluyen:

- **Agentes de logs**: fluentd, logstash, filebeat que recolectan logs de cada nodo y los envian a un sistema centralizado.
- **Monitoreo**: node-exporter (Prometheus), datadog-agent, collectd que exponen metricas del nodo.
- **Redes**: implementaciones de CNI como Calico, Weave o Cilium que gestionan la conectividad de red en cada nodo.
- **Almacenamiento**: drivers de CSI (Container Storage Interface) que montan sistemas de archivos en los nodos.

> Los DaemonSets omiten por defecto los nodos master debido a la taint `node-role.kubernetes.io/master`. Para ejecutarlos en masters, es necesario anadir la tolerancia correspondiente, como en el ejemplo anterior.

### Seleccion de nodos

Un DaemonSet puede restringirse a un subconjunto de nodos mediante `nodeSelector` o afinidades:

```yaml
spec:
  template:
    spec:
      nodeSelector:
        disk: ssd
```

De esta forma, solo los nodos etiquetados con `disk=ssd` ejecutaran el Pod. Esto es util para diferenciar cargas de trabajo entre maquinas con distinta capacidad.

## StatefulSet

Un StatefulSet gestiona aplicaciones con estado que requieren garantias sobre la identidad y el orden de los Pods. A diferencia de un Deployment, donde los Pods son intercambiables, en un StatefulSet cada Pod tiene una identidad unica y persistente.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

### Caracteristicas distintivas

**Identidad de red estable**: cada Pod recibe un nombre predecible: `<statefulset>-<ordinal>`. Para el ejemplo anterior: `mysql-0`, `mysql-1`, `mysql-2`. Estos nombres se resuelven mediante un Headless Service (definido con `clusterIP: None`), permitiendo que cada Pod tenga un DNS fijo: `mysql-0.mysql.default.svc.cluster.local`.

**Ordenamiento garantizado**: los Pods se crean secuencialmente de 0 a N-1, y cada uno debe estar listo antes de crear el siguiente. En la eliminacion, el orden se invierte: se elimina primero el Pod N-1, luego el N-2, etc. Este comportamiento es configurable con `podManagementPolicy: Parallel`.

**Almacenamiento persistente por Pod**: `volumeClaimTemplates` genera un PVC independiente para cada replica. Si `mysql-1` se elimina y se recrea, el nuevo Pod monta el mismo PVC que el original, preservando los datos. Los PVCs no se eliminan al eliminar el StatefulSet, lo que protege contra borrados accidentales.

### Casos de uso

- **Bases de datos relacionales**: MySQL, PostgreSQL con replicacion lider-seguidor.
- **Bases de datos NoSQL**: Cassandra, MongoDB, Elasticsearch, ZooKeeper.
- **Sistemas de colas**: Kafka, RabbitMQ, NATS.
- **Cualquier aplicacion que necesite identificadores unicos estables**: sistemas de votacion, servicios de locking distribuido.

> StatefulSet no gestiona la replicacion de datos ni la conmutacion por error. Sigue siendo responsabilidad de la aplicacion o de un operador especializado (como el MySQL Operator o el Cassandra Operator) manejar la consistencia y la recuperacion ante fallos.

### Actualizaciones en StatefulSets

Los StatefulSets soportan RollingUpdate con `partition`, que permite dividir la actualizacion en fases:

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 2
```

Con `partition: 2`, solo los Pods con ordinal >= 2 se actualizan. Los Pods 0 y 1 permanecen con la version anterior. Esto permite canary deployments y migraciones controladas en sistemas con estado.

