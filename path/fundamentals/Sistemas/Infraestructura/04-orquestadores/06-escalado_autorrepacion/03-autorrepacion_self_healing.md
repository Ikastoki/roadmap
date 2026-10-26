# Autorrepacion y Health Checks

La capacidad de un sistema para detectar y corregir automaticamente fallos sin intervención humana constituye la diferencia entre una infraestructura resiliente y una que requiere atencion constante. En Kubernetes, la autorrepacion se materializa a traves de dos mecanismos complementarios: los **health checks** (sondeos de salud) que permiten al sistema evaluar el estado de cada contenedor, y las **reacciones automaticas** del kubelet y el control plane que actuan cuando un contenedor o nodo falla. Esta combinacion crea un ciclo de vida autónomo donde los pods fallidos son reiniciados, los nodos enfermos son evacuados y las cargas de trabajo se migran para mantener la disponibilidad del servicio.

Los health checks en Kubernetes no son simplemente mecanismos de monitoreo pasivos: son contratos explicitos entre la aplicacion y el orquestador que definen bajo que condiciones un contenedor se considera funcional. Una aplicacion que no expone health checks claros es, para el orquestador, una caja negra incapaz de ser gestionada automaticamente. El primer paso hacia la autorrepacion efectiva consiste en definir con precision que significa "saludable" para cada componente del sistema.

## Liveness, Readiness y Startup Probes

Kubernetes distingue tres tipos de sondeos con responsabilidades bien delimitadas. El **Liveness Probe** determina si el contenedor esta vivo: si falla, el kubelet mata y reinicia el contenedor. Este probe es critico para detectar deadlocks, memory leaks y situaciones donde el proceso esta ejecutándose pero no puede realizar su trabajo. El **Readiness Probe** determina si el contenedor esta listo para recibir trafico: si falla, se elimina temporalmente del balanceo de carga del Service correspondiente, permitiendo que la aplicacion complete tareas de inicializacion o recuperacion sin perder peticiones. El **Startup Probe** protege a las aplicaciones con tiempos de arranque largos, impidiendo que los probes de liveness y readiness se evaluen antes de que la aplicacion haya terminado de inicializarse.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: microservice-api
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: microservice-api
  template:
    metadata:
      labels:
        app: microservice-api
    spec:
      containers:
      - name: api
        image: myregistry/api:v2.1
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "250m"
            memory: "512Mi"
          limits:
            cpu: "500m"
            memory: "1Gi"
        startupProbe:
          httpGet:
            path: /healthz/startup
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 30
        livenessProbe:
          httpGet:
            path: /healthz/liveness
            port: 8080
          initialDelaySeconds: 0
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /healthz/readiness
            port: 8080
          initialDelaySeconds: 0
          periodSeconds: 5
          timeoutSeconds: 2
          failureThreshold: 2
```

> El `failureThreshold` del `startupProbe` debe ser suficientemente alto para permitir que la aplicacion se inicialice incluso en condiciones adversas (cold start, carga inicial de modelos, conexiones a bases de datos). Un valor tipico es `30` con `periodSeconds: 5`, lo que permite hasta 150 segundos de arranque.

## Tipos de Sondas

Cada probe puede implementarse mediante cuatro metodos de verificacion distintos, seleccionados segun la naturaleza de la aplicacion. El metodo **HTTP** envia una peticion GET a un endpoint especifico y considera saludable cualquier respuesta con codigo 2xx o 3xx. El metodo **TCP** verifica que se pueda establecer una conexion TCP en el puerto indicado, sin necesidad de enviar datos. El metodo **Exec** ejecuta un comando dentro del contenedor y evalua su codigo de salida. El metodo **gRPC** utiliza el protocolo nativo de health checking de gRPC, enviando una solicitud `grpc.health.v1.Health/Check`.

```yaml
# metodo HTTP - para servicios REST o GraphQL
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
    httpHeaders:
    - name: X-Custom-Header
      value: healthcheck
  periodSeconds: 10

# metodo TCP - para bases de datos, colas de mensajes
livenessProbe:
  tcpSocket:
    port: 5432
  periodSeconds: 15

# metodo Exec - para aplicaciones sin endpoint HTTP
livenessProbe:
  exec:
    command:
    - /bin/sh
    - -c
    - "pgrep -x postgres || exit 1"
  periodSeconds: 10

# metodo gRPC - para servicios gRPC nativos
livenessProbe:
  grpc:
    port: 50051
  periodSeconds: 10
```

> El metodo **gRPC** es la forma mas elegante de health checking para servicios que implementan el protocolo gRPC, pero requiere que la aplicacion exponga el servicio `grpc.health.v1.Health` definido en el estandar [GRPC Health Checking Protocol](https://github.com/grpc/grpc/blob/master/doc/health-checking.md). La mayoria de los frameworks gRPC modernos (Go, Java, Python) incluyen implementaciones de este servicio.

## Reacciones Automaticas del Sistema

Mas alla de los health checks, Kubernetes implementa un conjunto de reacciones automaticas que conforman la capa de autorrepacion del cluster. Cuando un contenedor falla su liveness probe, el kubelet ejecuta un **restart** del contenedor preservando el mismo pod. Si el pod es eliminado, el controlador de replicacion crea automaticamente un nuevo pod para mantener el numero de replicas deseado. Cuando un nodo queda unreachable, el control plane detecta la falta de heartbeat y despues de un periodo configurable (`pod-eviction-timeout`, por defecto 5 minutos), marca los pods como `Terminating` y los reprograma en nodos saludables.

```bash
# verificacion de eventos de autorrepacion
kubectl get events -n production --sort-by='.lastTimestamp' | grep -E "Unhealthy|Killing|BackOff"

# inspeccion de un pod con problemas de salud
kubectl describe pod <pod-name> -n production

# observacion de reinicios en un pod
kubectl get pod <pod-name> -n production -o jsonpath='{.status.containerStatuses[*].restartCount}'

# revision de pods evictados por OOM
kubectl get pods -n production --field-selector=status.phase=Failed -o json | \
  jq '.items[] | select(.status.reason=="OOMKilled") | .metadata.name'

# verificacion de nodos con problemas
kubectl get nodes --show-labels | grep -E "Ready|NotReady"
kubectl describe node <node-name> | grep -A5 "Conditions"
```

> El `pod-eviction-timeout` controla cuanto tiempo espera el control plane antes de evictar los pods de un nodo no disponible. En clusters con alta disponibilidad, se recomienda reducir este valor a `2m` para acelerar la recuperacion, pero en entornos con conexiones de red inestables, un valor demasiado bajo puede provocar evicciones innecesarias.

## Pod Disruption Budgets y Budgets de Disponibilidad

Los **Pod Disruption Budgets** (PDBs) complementan el mecanismo de autorrepacion al definir restricciones sobre cuantos pods pueden estar simultaneamente indisponibles durante interrupciones planificadas o no planificadas. Un PDB garantiza que, incluso durante mantenciones de nodos o reinicios de pods, el numero minimo de instancias operativas se mantenga por encima del umbral definido. Esto es especialmente critico para servicios de mision critica donde la disponibilidad del 99.99% requiere que siempre existan al menos N replicas funcionando.

```yaml
# garantiza que al menos 2 replicas esten disponibles
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: microservice-api

---
# alternativa: permite maximo 1 pod no disponible
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: worker-pdb
  namespace: production
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: background-worker
```

> `minAvailable` y `maxUnavailable` son mutuamente excluyentes: solo se puede definir uno de los dos en un PDB. `minAvailable` es mas intuitivo cuando se conoce el numero exacto de replicas necesarias, mientras que `maxUnavailable` es mas flexible cuando el numero de replicas puede variar.

