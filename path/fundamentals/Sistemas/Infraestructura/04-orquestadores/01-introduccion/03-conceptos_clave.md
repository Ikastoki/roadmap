# Conceptos clave

Para trabajar con cualquier orquestador de contenedores es imprescindible comprender el vocabulario fundamental que define su modelo operativo. Estos conceptos no son terminos aislados; forman un entramado interconectado donde cada pieza depende de las demas. Un clúster se compone de nodos, los nodos ejecutan unidades de ejecucion, las unidades de ejecucion pertenecen a servicios, y los servicios se comunican entre si a traves de mecanismos de descubrimiento y balanceo de carga. Comprender esta cadena es el punto de partida para diseñar sistemas distribuidos fiables sobre cualquier plataforma de orquestacion.

Aunque cada orquestador utiliza terminologia propia, los conceptos subyacentes son equivalentes en la mayoria de plataformas. Kubernetes emplea Pods, Services e Ingress; Docker Swarm utiliza Tasks y Services; Nomad trabaja con Tasks, Groups y Allocations. A continuacion se desglosan los conceptos universales con referencias especificas a las plataformas mas relevantes.

## Cluster

Un clúster es el conjunto de servidores (nodos) que el orquestador gestiona como una unidad coherente de computacion. Desde la perspectiva del usuario, el clúster se presenta como un recurso unico con capacidad agregada de CPU, memoria y almacenamiento, independientemente de cuantos servidores lo componen. Todos los nodos del clúster se comunican entre si a traves de una red interna y comparten el estado del sistema a traves del plano de control.

```bash
# Ver los nodos de un clúster Kubernetes
kubectl get nodes

# Salida ejemplo:
# NAME     STATUS   ROLES           AGE   VERSION
# node-1   Ready    control-plane   30d   v1.29.2
# node-2   Ready    <none>          30d   v1.29.2
# node-3   Ready    <none>          30d   v1.29.2

# En Docker Swarm
docker node ls

# ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS
# abc123 *   node-1      Ready     Active       Leader
# def456     node-2      Ready     Active
# ghi789     node-3      Ready     Active
```

## Nodos: Control Plane y Worker

Los nodos se dividen en dos categorias funcionales que cumplen roles claramente diferenciados dentro del clúster.

**Nodos del plano de control (Control Plane):** alojan los componentes que toman decisiones sobre el estado del sistema: el servidor API (punto de entrada para todas las operaciones), el scheduler (que decide donde ejecutar cada contenedor), el controller manager (que ejecuta los bucles de conciliacion) y, en plataformas como Kubernetes, el almacén de estado distribuido (etcd). Estos nodos no ejecutan workloads de usuario; exclusivamente administran el clúster.

**Nodos de trabajo (Worker):** ejecutan las unidades de ejecucion que componen las aplicaciones. Cada nodo de trabajo ejecuta un agente (kubelet en Kubernetes, Docker daemon en Swarm) que se comunica con el plano de control y gestiona los contenedores en su maquina. Son los nodos donde se consumen los recursos de CPU y memoria para ejecutar servicios reales.

```yaml
# Asignar un pod a un nodo especifico por nombre
apiVersion: v1
kind: Pod
metadata:
  name: base-datos
spec:
  containers:
  - name: postgres
    image: postgres:16
  nodeSelector:
    kubernetes.io/hostname: node-3

# O usar taints y tolerancias para reservar nodos
# En el nodo-3:
# kubectl taint nodes node-3 dedicated=database:NoSchedule
```

> En clústers pequenos de desarrollo, es comun ejecutar el plano de control y los workloads en el mismo nodo. En produccion, se recomienda separarlos fisicamente para aislar fallos y garantizar la disponibilidad del plano de control ante caidas de nodos de trabajo.

## Unidad de ejecucion

La unidad de ejecucion es la abstraccion minima que el orquestador programa en un nodo. Cada plataforma la nombra y modela de forma diferente, pero el concepto es equivalente: un contenedor o grupo de contenedores que comparten contexto de ejecucion.

**Kubernetes - Pod:** la unidad atomica de despliegue. Un Pod contiene uno o varios contenedores que comparten red (misma IP, mismos puertos), almacenamiento y contexto de ejecucion. Los contenedores dentro de un Pod se comunican a traves de localhost.

**Docker Swarm - Task:** representa una instancia de un servicio en un nodo. Cada task ejecuta un contenedor y es asignada a un nodo especifico por el scheduler.

**Nomad - Allocation:** la unidad basica de despliegue, equivalente a un Pod. Puede contener multiples tasks que comparten recursos en el mismo nodo.

```yaml
# Pod con contenedor sidecar para logs
apiVersion: v1
kind: Pod
metadata:
  name: api-con-sidecar
spec:
  containers:
  - name: api
    image: myapp-api:2.1.0
    ports:
    - containerPort: 8080
  - name: log-collector
    image: fluentd:latest
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
  volumes:
  - name: logs
    emptyDir: {}
```

## Service

Un Service es una abstraccion que define un conjunto logico de Pods (o tasks) y una politica de acceso a ellos. Proporciona una IP estable y un nombre DNS que no cambian aunque los contenedores subyacentes se creen, destruyan o reubiquen. El Service actua como punto de encuentro estable en un entorno inherentemente inestable.

```yaml
# Service en Kubernetes que expone la API internamente
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP

# Ingress que expone el servicio externamente
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: api.miapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

## Descubrimiento de servicios

El descubrimiento de servicios es el mecanismo que permite que un servicio localice a otro dentro del clúster sin conocer su ubicacion fisica ni su IP. Cada orquestador implementa su propio sistema de registro y resolucion de nombres, generalmente basado en DNS interno. Cuando un servicio se registra, el orquestador crea automaticamente un registro DNS apuntando a todas las instancias activas de ese servicio.

```bash
# En Kubernetes, un servicio se resuelve por su nombre
# Dentro de un pod, este comando funciona:
nslookup api-service

# Salida:
# Name:      api-service
# Address:   10.96.45.123  (IP virtual del Service, no las IPs de los pods)

# En Docker Swarm, el DNS integrado resuelve servicios por nombre
# Desde dentro de un contenedor en la misma red:
nslookup api
```

## Load Balancer

El balanceador de carga distribuye el trafico entrante entre las multiples instancias de un servicio. Puede operar en diferentes capas del modelo OSI: L4 (TCP/UDP, mas rapido pero menos flexible) o L7 (HTTP, con capacidad de enrutamiento basado en contenido). En Kubernetes, el tipo de Service `LoadBalancer` integra un balanceador de carga externo del proveedor de nube, mientras que `ClusterIP` ofrece balanceo interno. Los Ingress Controllers proporcionan balanceo L7 con funcionalidades avanzadas como TLS termination, rate limiting y routing basado en paths.

## Configuracion y secretos

Los orquestadores proporcionan mecanismos para inyectar configuracion y credenciales en los contenedores sin incluirlos en las imagenes. Esto separa la configuracion del codigo y permite cambiar parametros sin reconstruir imagenes. En Kubernetes, los objetos ConfigMap almacenan configuracion no sensible (variables de entorno, archivos de configuracion) y los Secrets almacenan datos sensibles (contraseñas, tokens, certificados) con cifrado basico.

```yaml
# ConfigMap para variables de configuracion
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
data:
  DB_HOST: "postgres-service"
  DB_PORT: "5432"
  LOG_LEVEL: "info"
  app.properties: |
    retry.attempts=3
    timeout.seconds=30

---
# Secret para credenciales
apiVersion: v1
kind: Secret
metadata:
  name: api-secrets
type: Opaque
data:
  DB_PASSWORD: cG9zdGdyZXMxMjM=  # base64 de "postgres123"
  API_KEY: c2VjcmV0LWFwaS1rZXk=

---
# Uso en un Pod
apiVersion: v1
kind: Pod
metadata:
  name: api
spec:
  containers:
  - name: api
    image: myapp-api:2.1.0
    envFrom:
    - configMapRef:
        name: api-config
    - secretRef:
        name: api-secrets
```

> Los Secrets de Kubernetes solo cifran en reposo por defecto (y no en todas las configuraciones). Para un manejo seguro de credenciales en produccion, se recomienda integrar soluciones como Vault de HashiCorp o los Secrets Manager de los proveedores de nube.

## Namespace y Project

Los namespaces (o projects en OpenShift/Rancher) son mecanismos de aislamiento logico dentro de un clúster. Permiten dividir el cluster en regiones virtuales con politicas de acceso, limites de recursos y configuraciones independientes. Son especialmente utiles en entornos multi-tenant donde diferentes equipos o proyectos comparten la misma infraestructura pero necesitan aislamiento.

```bash
# Crear un namespace
kubectl create namespace equipo-backend

# Desplegar un servicio en un namespace especifico
kubectl apply -f deployment.yaml -n equipo-backend

# Ver recursos de un namespace
kubectl get all -n equipo-backend

# Limitar recursos del namespace mediante ResourceQuota
```

```yaml
# ResourceQuota para limitar el consumo de un namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota-backend
  namespace: equipo-backend
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    pods: "20"
```

## Voluepnes persistentes

Los voluepnes persistentes (Persistent Volumes en Kubernetes, Volume Drivers en Docker) permiten que los contenedores almacenen datos que sobreviven a su ciclo de vida. A diferencia de los voluepnes efimeros que se destruyen con el contenedor, los voluepnes persistentes se gestionan de forma independiente y pueden ser reasignados a nuevos contenedores cuando los anteriores terminan.

```yaml
# PersistentVolumeClaim en Kubernetes
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: datos-postgres
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: fast-ssd

---
# Uso en un StatefulSet para base de datos
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:16
        volumeMounts:
        - name: datos
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: datos
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

## Agente

El agente es el componente que se ejecuta en cada nodo de trabajo y se comunica con el plano de control. Recibe instrucciones sobre que contenedores ejecutar, monitorea su estado y reporta metricas de salud y recursos al plano de control. En Kubernetes, el agente es kubelet; en Docker Swarm, es el daemon Docker con el modulo swarm; en Nomad, es el cliente Nomad. Sin el agente activo, el nodo queda marcado como `NotReady` o `down` y el orquestador reubica los contenedores que ahi se ejecutaban en otros nodos disponibles.

## CLI y API

Toda interaccion con el orquestador se realiza a traves de dos interfaces: la CLI (interfaz de linea de comandos) para operaciones interactivas y scripts, y la API REST para integracion programatica y herramientas automatizadas. La API es el contrato formal entre el cliente y el orquestador; todas las operaciones de la CLI son, en realidad, llamadas HTTP a esta API.

```bash
# CLI de Kubernetes
kubectl get pods -n production
kubectl scale deployment api --replicas=5
kubectl logs -f deployment/api -n production

# Equivalente con la API directa
curl -k https://kubernetes-api:6443/api/v1/namespaces/production/pods \
  -H "Authorization: Bearer $TOKEN"

# Docker Swarm CLI
docker service ls
docker service scale api=5
docker service logs api
```

