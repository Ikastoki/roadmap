# Patrones de orquestacion

Los patrones de orquestacion son soluciones reutilizables a problemas recurrentes que surgen al gestionar aplicaciones en entornos distribuidos. No se trata de arquitecturas de la aplicacion en si, sino de patrones de infraestructura que el orquestador facilita o hace posibles. Estos patrones aprovechan la capacidad del orquestador para co-localizar contenedores, compartir redes y recursos, y gestionar ciclos de vida coordinados entre multiples procesos.

Cada patron resuelve un problema especifico: desde inyectar funcionalidad auxiliar sin modificar la imagen principal, hasta coordinar eleccion de lider en sistemas distribuidos. La comprension de estos patrones permite disenar soluciones robustas sin reinventar la rueda en cada despliegue.

## Sidecar

El patron sidecar adjunta un contenedor auxiliar al contenedor principal dentro del mismo pod. Ambos comparten red (`localhost`), almacenamiento y ciclo de vida del pod. El sidecar no es parte de la logica de negocio, sino que provee funcionalidad transversal: recoleccion de logs, proxy de red, monitoreo, sincronizacion de datos, inyeccion de secretos.

La principal ventaja del sidecar es la separacion de responsabilidades. El contenedor principal no necesita conocer la existencia del sidecar; simplemente expone su servicio a traves de un puerto o escribe logs en un directorio compartido, y el sidecar se encarga del transporte, filtrado o transformacion de esos datos.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-con-logging
spec:
  replicas: 2
  selector:
    matchLabels:
      app: mi-aplicacion
  template:
    metadata:
      labels:
        app: mi-aplicacion
    spec:
      containers:
        - name: app
          image: mi-org/app:v2.1.0
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: shared-logs
              mountPath: /var/log/app
        - name: fluentbit-sidecar
          image: fluent/fluent-bit:2.1
          volumeMounts:
            - name: shared-logs
              mountPath: /var/log/app
              readOnly: true
            - name: fluent-config
              mountPath: /fluent-bit/etc/
          resources:
            limits:
              cpu: 100m
              memory: 64Mi
      volumes:
        - name: shared-logs
          emptyDir: {}
        - name: fluent-config
          configMap:
            name: fluentbit-config
```

> El sidecar consume recursos del pod. Si la funcionalidad auxiliar no es estrictamente necesaria en cada instancia, considera ejecutarla como un DaemonSet o servicio separado.

## Ambassador

El patron ambassador extiende el sidecar con un proposito especifico: actuar como proxy hacia servicios externos al cluster. El contenedor principal se comunica con un endpoint local (`localhost`), y el ambassador se encarga del enrutamiento, balanceo de carga, reconexion automatica y manejo de circuitos hacia el servicio remoto.

Este patron es especialmente util cuando el servicio externo no puede ser expuesto directamente en la red del cluster (por restricciones de firewall, autenticacion o protocolos no HTTP). El ambassador encapsula toda la complejidad de la comunicacion remota.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-con-ambassador
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app-ambassador
  template:
    metadata:
      labels:
        app: app-ambassador
    spec:
      containers:
        - name: app
          image: mi-org/app:v1.0
          env:
            - name: EXTERNAL_DB_HOST
              value: "localhost"
            - name: EXTERNAL_DB_PORT
              value: "5432"
        - name: ambassador
          image: envoyproxy/envoy:v1.28
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: envoy-config
              mountPath: /etc/envoy/
      volumes:
        - name: envoy-config
          configMap:
            name: envoy-ambassador-config
```

```yaml
# ConfigMap de Envoy como ambassador
apiVersion: v1
kind: ConfigMap
metadata:
  name: envoy-ambassador-config
data:
  envoy.yaml: |
    static_resources:
      listeners:
        - name: db_listener
          address:
            socket_address:
              address: 0.0.0.0
              port_value: 5432
          filter_chains:
            - filters:
                - name: envoy.filters.network.tcp_proxy
                  typed_config:
                    "@type": type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy
                    stat_prefix: db_proxy
                    cluster: remote_db
                    idle_timeout: 60s
      clusters:
        - name: remote_db
          type: STRICT_DNS
          lb_policy: ROUND_ROBIN
          load_assignment:
            cluster_name: remote_db
            endpoints:
              - lb_endpoints:
                  - endpoint:
                      address:
                        socket_address:
                          address: db-externa.ejemplo.com
                          port_value: 5432
          connect_timeout: 5s
          tcp_keepalive:
            keepalive_time: 300s
```

## Init Containers

Los init containers se ejecutan antes de que el contenedor principal inicie. Cada init container debe completarse exitosamente antes de que arranque el siguiente. Son ideales para tareas de inicializacion que requieren condiciones que no se cumplen al momento del arranque: esperar a que un servicio externo este disponible, descargar archivos de configuracion, generar certificados, o ejecutar migraciones de esquema.

A diferencia de los sidecars, los init containers no persisten durante la vida del pod; terminan su trabajo y desaparecen. Esto los hace ligeros y especializados en tareas de un solo uso.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-con-init
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-init
  template:
    metadata:
      labels:
        app: app-init
    spec:
      initContainers:
        - name: wait-for-db
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              until nc -z postgres-service 5432; do
                echo "Esperando a que PostgreSQL este disponible..."
                sleep 2
              done
              echo "PostgreSQL listo"
        - name: run-migrations
          image: mi-org/app:v1.0
          command:
            - python
            - -m
            - alembic
            - upgrade
            - head
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: url
        - name: download-config
          image: curlimages/curl:8.4.0
          command:
            - sh
            - -c
            - curl -o /config/features.json https://config-service.internal/features.json
          volumeMounts:
            - name: config-volume
              mountPath: /config
      containers:
        - name: app
          image: mi-org/app:v1.0
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: config-volume
              mountPath: /app/config
              readOnly: true
      volumes:
        - name: config-volume
          emptyDir: {}
```

> Si un init container falla, el orquestador reinicia el pod de acuerdo con la politica `restartPolicy`. En entornos de produccion, es recomendable implementar mecanismos de reintentos limitados dentro del propio init container para evitar ciclos infinitos de reinicio.

## Leader Election

En sistemas distribuidos con multiples replicas, frecuentemente es necesario elegir un unico nodo como "lider" que coordine tareas que no pueden ejecutarse en paralelo: generacion de reportes periodicos, ejecucion de cronjobs, actualizacion de caches globales, o coordinacion de barreras. El patron leader election resuelve este problema utilizando primitivas de consensus del orquestador.

Kubernetes provee primitive para leader election a traves de `coordination.k8s.io/v1` (Leases). Un candidato crea o actualiza un Lease; el que obtiene el campo `holderIdentity` es el lider. Los demas quedan en modo standby, listos para asumir el liderazgo si el lider actual falla.

```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: app-leader
  namespace: production
spec:
  holderIdentity: "app-pod-abc123"
  leaseDurationSeconds: 15
  acquireTime: "2025-07-26T10:00:00Z"
  renewTime: "2025-07-26T10:00:15Z"
  leaseTransitions: 0
```

```python
# Ejemplo de leader election en Python usando la libreria de Kubernetes
from kubernetes import client, config
import datetime
import os

config.load_incluster_config()
coordination = client.CoordinationV1Api()

NAMESPACE = os.environ.get("POD_NAMESPACE", "default")
LEASE_NAME = "app-leader"
HOLDER_ID = os.environ.get("POD_NAME", "unknown")
INTERVAL = 5  # segundos

def try_acquire():
    try:
        lease = coordination.read_namespaced_lease(LEASE_NAME, NAMESPACE)
        if lease.spec.holder_identity == HOLDER_ID:
            renew_lease(lease)
            return True
        if lease.spec.renew_time + datetime.timedelta(seconds=lease.spec.lease_duration_seconds) < datetime.datetime.utcnow():
            lease.spec.holder_identity = HOLDER_ID
            lease.spec.acquire_time = datetime.datetime.utcnow()
            lease.spec.renew_time = datetime.datetime.utcnow()
            coordination.replace_namespaced_lease(LEASE_NAME, NAMESPACE, lease)
            return True
        return False
    except client.exceptions.ApiException as e:
        if e.status == 404:
            create_lease()
            return True
        raise

def create_lease():
    lease = client.V1Lease(
        metadata=client.V1ObjectMeta(name=LEASE_NAME, namespace=NAMESPACE),
        spec=client.V1LeaseSpec(
            holder_identity=HOLDER_ID,
            lease_duration_seconds=15,
            acquire_time=datetime.datetime.utcnow(),
            renew_time=datetime.datetime.utcnow()
        )
    )
    coordination.create_namespaced_lease(NAMESPACE, lease)

def renew_lease(lease):
    lease.spec.renew_time = datetime.datetime.utcnow()
    coordination.replace_namespaced_lease(LEASE_NAME, NAMESPACE, lease)
```

> La eleccion de lider es fundamental en aplicaciones que manejan tareas batch o cron. Sin este patron, multiples replicas ejecutarian simultaneamente la misma tarea, causando duplicados o corrupcion de datos.

