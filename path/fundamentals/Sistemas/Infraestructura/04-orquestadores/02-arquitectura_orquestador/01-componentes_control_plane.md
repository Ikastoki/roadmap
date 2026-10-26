# Componentes del Control Plane

El control plane es el cerebro de cualquier sistema de orquestacion de contenedores. Representa el conjunto de procesos coordinadores que mantienen el estado global del cluster, toman decisiones de programacion y ejecutan acciones correctivas para que la realidad del sistema coincida con la intencion declarada por el operador. Cada orquestador — Kubernetes, Nomad, Docker Swarm — implementa estos componentes con variaciones en su arquitectura interna, pero todos convergen en los mismos roles funcionales: un punto de entrada centralizado para operaciones, un almacen distribuido de estado, un planificador de recursos y un gestor de controladores que aplica la logica de autorreparacion y escalado.

La separacion entre el control plane y los nodos worker es la decision arquitectonica mas significativa de los orquestadores modernos. Esta division permite que el sistema de toma de decisiones opere de forma independiente a donde se ejecutan realmente los contenedores, lo que habilita tolerancia a fallos en ambos lados de la arquitectura. Un nodo worker puede fallar completamente sin que el control plane pierda capacidad de decision, y viceversa.

```
┌─────────────────────────────────────────────────────────┐
│                     CONTROL PLANE                        │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │  API Server  │  │  Scheduler   │  │  Controller   │  │
│  │  (Entry)     │  │  (Planner)   │  │  Manager      │  │
│  └──────┬───────┘  └──────┬───────┘  └───────┬───────┘  │
│         │                 │                   │          │
│         └────────┬────────┴───────────────────┘          │
│                  │                                       │
│         ┌────────▼────────┐                              │
│         │  etcd / Consul  │                              │
│         │  (State DB)     │                              │
│         └─────────────────┘                              │
└─────────────────────┬───────────────────────────────────┘
                      │  mTLS
         ┌────────────┼────────────┐
         │            │            │
    ┌────▼────┐  ┌────▼────┐  ┌────▼────┐
    │ Worker  │  │ Worker  │  │ Worker  │
    │ Node 1  │  │ Node 2  │  │ Node 3  │
    └─────────┘  └─────────┘  └─────────┘
```

## API Server

El API Server es el unico componente del control plane que expone una interfaz externa. En Kubernetes se denomina `kube-apiserver`, en Nomad es el `nomad agent -server`, y en Docker Swarm el `swarm manager` actua como punto de entrada unico. Toda operacion — crear un pod, escalar un servicio, consultar metricas, eliminar un recurso — debe pasar obligatoriamente por este componente. No existe ningun camino alternativo para interactuar con el estado del cluster.

Esta centralizacion tiene propositos de seguridad y consistencia. El API Server implementa autenticacion (quien eres), autorizacion (que puedes hacer) y validacion (los datos que envias son correctos) antes de que cualquier cambio llegue al almacen de estado. Ademas, actua como un endpoint de la API REST que los clientes consumen de forma programatica.

En Kubernetes, el API Server soporta multiples mecanismos de autenticacion simultaneamente:

```yaml
# Ejemplo de configuracion de kube-apiserver
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
    - name: kube-apiserver
      command:
        - kube-apiserver
        - --advertise-address=10.0.1.10
        - --authorization-mode=Node,RBAC
        - --client-ca-file=/etc/kubernetes/pki/ca.crt
        - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
        - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
        - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
        - --secure-port=6443
        - --service-account-key-file=/etc/kubernetes/pki/sa.pub
        - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
        - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
```

> En un cluster de produccion, el API Server debe ejecutarse al menos en tres nodos del control plane para garantizar disponibilidad. Los balanceadores de carga distribuyen las peticiones entre las instancias, pero todas las operaciones de escritura convergen al mismo lider a traves de etcd.

## Base de datos de estado (etcd)

etcd es la memoria del cluster. Almacena toda la informacion sobre nodos, pods, servicios, configuraciones, secretos y el estado actual de cada recurso orquestado. No es solo una base de datos pasiva: almacena tanto el estado deseado como el estado real, lo que permite al control plane calcular los deltas necesarios para alcanzar la convergencia.

etcd implementa el consenso distribuido Raft, que garantiza que todas las instancias de etcd en el cluster mantengan datos identicos. En un cluster con cinco nodos de etcd, al menos tres deben estar operativos para aceptar escrituras (quorum de 2n+1). Este mecanismo protege contra perdida de datos incluso si la mayoria de los nodos de almacen falla simultaneamente.

```
┌─────────── etcd Cluster (5 nodos) ───────────┐
│                                                │
│   ┌────────┐  ┌────────┐  ┌────────┐         │
│   │ Node 1 │  │ Node 2 │  │ Node 3 │  Líder  │
│   │ (Follower)│ (Follower)│ (Follower)│◄─Líder  │
│   └────────┘  └────────┘  └────────┘         │
│   ┌────────┐  ┌────────┐                      │
│   │ Node 4 │  │ Node 5 │                      │
│   │ (Follower)│ (Follower)│                      │
│   └────────┘  └────────┘                      │
│                                                │
│   Quorum = (5/2) + 1 = 3 nodos mínimos        │
│   Tolerancia a fallos = (5-1)/2 = 2 nodos     │
└────────────────────────────────────────────────┘
```

```bash
# Consultar el estado de etcd en un cluster de Kubernetes
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

# Ver todos los pods almacenados
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get / --prefix --keys-only | head -50
```

> Alternativas a etcd incluyen Consul (Hashicorp, usado por Nomad), ZooKeeper (Apache, usado por Kafka y Hadoop) y DynamoDB (AWS, usado por EKS). Todos implementan consenso distribuido pero con protocolos diferentes: Consul usa Raft como etcd, ZooKeeper usa ZAB, y DynamoDB ofrece consistencia eventual configurable.

## Scheduler / Planificador

El scheduler (planificador) es el componente responsable de asignar cada unidad de trabajo a un nodo del cluster. No toma decisiones sobre que ejecutar ni como configurar los contenedores: simplemente recibe peticiones pendientes de asignacion y evalua en cual de los nodos disponibles tienen mayor probabilidad de ejecutarse con exito. En Kubernetes este componente se llama `kube-scheduler`, en Nomad es el `scheduler` embebido en el servidor, y en Swarm la decision la toma el `dispatcher` dentro del manager.

El proceso de planificacion implica multiples fases. Primero, el scheduler filtra los nodos que no cumplen los requisitos minimos: suficiente memoria, arquitectura de CPU compatible, tolerancias de hardware. Luego aplica politicas de afinidad y anti-afinidad para decidir que nodos son preferibles. Finalmente, asigna un score numerico a cada candidato y selecciona el de mayor puntuacion.

```yaml
# Ejemplo de restricciones de planificacion en Kubernetes
apiVersion: apps/v1
kind: Deployment
metadata:
  name: base-de-datos
spec:
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      # Nodos que deben tener un label especifico
      nodeSelector:
        disktype: ssd
        region: eu-west-1
      # Reglas de afinidad mas granulares
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: kubernetes.io/os
                    operator: In
                    values:
                      - linux
        # No colocar dos pods en el mismo nodo
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - postgres
                topologyKey: kubernetes.io/hostname
      containers:
        - name: postgres
          image: postgres:16
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2"
              memory: "4Gi"
```

> El scheduler no ejecuta nada directamente. Solo escribe una decision en etcd: "este pod debe ejecutarse en ese nodo". El agente del nodo destino es quien posteriormente recibe la instruccion y ejecuta el contenedor.

## Controller Manager

El Controller Manager implementa el ciclo de control fundamental de los orquestadores: observar el estado actual, compararlo con el estado deseado, y ejecutar las acciones necesarias para reducir la diferencia. En Kubernetes, el `kube-controller-manager` ejecuta docenas de controladores especializados, cada uno responsable de un tipo de recurso. En Nomad, esta logica se distribuye entre el scheduler y los agentes de los nodos.

Cada controlador opera de forma independiente en un bucle infinito. El controlador de replicas monitorea cuantos pods estan ejecutandose realmente vs los solicitados. Si un pod muere, detecta el deficit y crea uno nuevo. El controlador de nodos monitora el heartbeat de cada nodo y lo marca como `NotReady` si deja de responder. El controlador de endpoints actualiza las tablas de enrutamiento cuando los pods cambian de IP.

```
┌─────── Ciclo de Control ───────┐
│                                │
│   Estado Deseado (etcd)        │
│          │                     │
│          ▼                     │
│   ┌──────────────┐            │
│   │ Comparar     │            │
│   │ Estado       │            │
│   └──────┬───────┘            │
│          │                     │
│          ▼                     │
│   ┌──────────────┐   No   ┌───┴────┐
│   │  Diferente?  │───────►│ Esperar│
│   └──────┬───────┘        └────────┘
│          │ Sí                    │
│          ▼                       │
│   ┌──────────────┐              │
│   │ Ejecutar     │              │
│   │ Acción       │              │
│   └──────┬───────┘              │
│          │                       │
│          ▼                       │
│   Estado Real (Nodo)             │
│          │                       │
│          └───────► Volver a inicio│
└─────────────────────────────────┘
```

```bash
# Ver los controladores activos en Kubernetes
kubectl get controllers -n kube-system

# Ver los eventos de un controlador especifico
kubectl get events --field-selector reason=ScalingReplicaSet --sort-by='.lastTimestamp'

# En Nomad, ver el estado del planificador
nomad job status <job-name>
nomad eval status <eval-id>
```

```yaml
# Ejemplo de HorizontalPodAutoscaler en Kubernetes
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-scaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 3
  maxReplicas: 50
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 120
```

> El patron de "reconciliacion" (reconciliation) es el nucleo de todo orquestador. En lugar de ejecutar acciones imperativas ("crea un contenedor"), el sistema compara constantemente el estado real con el deseado y actua para converger. Este modelo transforma la gestion de infraestructura en un sistema autoestabilizado que responde a fallos, cambios de carga y actualizaciones de configuracion sin intervencion manual.

