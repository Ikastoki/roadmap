# Modelos de Comunicacion

Los orquestadores de contenedores utilizan distintos modelos de comunicacion entre sus componentes, y la eleccion de uno u otro impacta directamente en la latencia de decisiones, la tolerancia a fallos de red y la escalabilidad del cluster. Los tres patrones predominantes son pull (el cliente solicita estado), push (el servidor envia instrucciones) y gossip (comunicacion peer-to-peer entre nodos). La mayoria de orquestadores modernos combinan los tres en diferentes proporciones segun el tipo de informacion que se transmite.

La distincion entre estos modelos no es meramente teorica. Un sistema que depende exclusivamente de push puede saturar el servidor central si hay miles de nodos que reportan simultaneamente. Un sistema puramente pull introduce latencia porque los clientes no reciben actualizaciones hasta su proximo ciclo de consulta. Gossip resuelve ambos problemas para ciertos tipos de informacion (descubrimiento, estado de salud) pero no es adecuado para instrucciones criticas que requieren consistencia inmediata.

```
┌───────────── Pull ──────────────┐
│                                  │
│  Worker ──"¿Qué debo hacer?"──► │
│  Worker ◄──"Ejecuta esto"────   │
│                                  │
│  Periodico, el cliente inicia    │
└──────────────────────────────────┘

┌───────────── Push ──────────────┐
│                                  │
│  Control Plane ──"Ejecuta esto"─►│
│  Control Plane ◄──"Hecho"────── │
│                                  │
│  El servidor inicia, inmediato   │
└──────────────────────────────────┘

┌─────── Gossip (Peer-to-Peer) ──────┐
│                                      │
│  Node A ◄──"Nodo C está caído"──►  │
│  Node A ──"Nodo C está caído"──►   │
│  Node B ◄──"Nodo C está caído"──   │
│                                      │
│  Diseminación gradual, sin líder     │
└──────────────────────────────────────┘
```

## Modelo Pull

En el modelo pull, son los clientes (agentes en los workers) quienes inician la comunicacion. Periodicamente envian peticiones al control plane preguntando si hay nuevas instrucciones o si el estado ha cambiado. Este patron es dominante en Kubernetes, donde el kubelet establece un watch long-polling contra el API Server para recibir actualizaciones de los pods asignados a su nodo.

La ventaja principal del pull es que el control plane no necesita mantener estado sobre la conectividad de cada agente. Si un agente se desconecta y vuelve, simplemente reanuda su ciclo de consulta desde donde lo dejo. Ademas, el agente puede adaptar la frecuencia de consulta segun su carga: en condiciones normales consulta cada 10 segundos, pero si detecta que esta sobrecargado puede espaciar las consultas.

```yaml
# kubelet configura su ciclo de pull via nodeStatusUpdateFrequency
# Valor por defecto: 10 segundos
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
nodeStatusUpdateFrequency: "10s"

# El kubelet tambien usa un "watch" contra el API Server
# para recibir cambios en pods asignados de forma masiva
# en lugar de consultar individualmente cada pod
```

```bash
# Observar los logs del kubelet mostrando ciclos de pull
journalctl -u kubelet -f | grep -E "SyncLoop|status"

# En Nomad, el cliente hace pull a los servidores cada 30 segundos por defecto
# Configuracion ajustable:
nomad agent -client \
  -retry-interval="10s" \
  -retry-max-interval="60s"

# Ver los ciclos de reconciliacion del kubelet
kubectl logs -n kube-system <kubelet-pod> --prefix | grep -i reconcil
```

```hcl
# En Nomad, el pull del cliente se configura en el bloque client
client {
  enabled = true
  
  # Intervalo de heartbeats al servidor
  heartbeat {
    interval    = "15s"
    threshold   = "10"
  }
  
  # Intervalo de sincronizacion de estado
  state_dir    = "/var/lib/nomad"
  alloc_dir    = "/opt/nomad/alloc"
  
  # Maximo de allocation groups que el cliente puede ejecutar
  max_allocs   = 100
}
```

> El modelo pull introduce un trade-off: la latencia. Si el control plane actualiza el estado de un pod, el kubelet no lo sabra hasta su proximo ciclo de poll (10s por defecto en Kubernetes). Para reducir esta latencia, Kubernetes usa websockets/long-polling que permiten al API Server notificar cambios inmediatamente al agente, convirtiendo el pull pasivo en un push ligero sobre la conexion persistente.

## Modelo Push

En el modelo push, es el control plane quien inicia la comunicacion, enviando instrucciones directamente a los agentes sin esperar a que estos las soliciten. Docker Swarm utiliza este modelo de forma predominante: el dispatcher del manager mantiene conexiones persistentes (gRPC streaming) con cada agente y le envia actualizaciones de las tareas asignadas de forma inmediata.

El push ofrece latencia minima entre la decision del control plane y su ejecucion. Cuando el scheduler asigna un pod a un nodo, la instruccion llega al agente en milisegundos. Sin embargo, este modelo requiere que el control plane mantenga un registro activo de todas las conexiones, lo que añade complejidad en terminos de gestion de memoria y manejo de reconexiones cuando los agentes pierden conectividad.

```
┌─────────── Push con conexiones persistentes ────────┐
│                                                      │
│  Control Plane (Swarm Manager)                      │
│       │                                              │
│       ├── gRPC Stream ──► Worker 1                  │
│       │   "Ejecuta nginx:v3"                         │
│       │                                              │
│       ├── gRPC Stream ──► Worker 2                  │
│       │   "Ejecuta redis:7"                          │
│       │                                              │
│       └── gRPC Stream ──► Worker 3                  │
│           "Elimina tarea X"                          │
│                                                      │
│  Si un stream se corta:                              │
│  - El agente reconecta y solicita estado completo    │
│  - El manager reenvia instrucciones pendientes       │
│                                                      │
└──────────────────────────────────────────────────────┘
```

```bash
# Docker Swarm usa push: el manager envia instrucciones a los agents
# Observar la comunicacion manager-agent
docker service logs <service-name> --follow

# En Swarm, el dispatcher asigna tareas via gRPC
# Ver los managers activos
docker node ls

# Ver las tareas asignadas a un nodo especifico
docker ps --filter "label=com.docker.swarm.task.id=<task-id>"

# Nomad tambien usa push para asignaciones criticas
# El planificador envia evaluciones directamente al agente
nomad alloc status <alloc-id> -json | jq '.ClientStatus'
```

```hcl
# Configuracion de Docker Swarm con timeout para push
# En /etc/docker/daemon.json
{
  "experimental": false,
  "metrics-addr": "0.0.0.0:9323",
  "features": {
    "buildkit": true
  },
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}

# Configuracion del agente Swarm en cada nodo
# via /etc/docker/daemon.json o flags del servicio docker
# La reconexion del agente tras caida del manager se gestiona
# automaticamente con exponential backoff
```

> Swarm combina push y pull: el dispatcher envia tareas nuevas via push, pero el agente hace pull periodico de su estado asignado como fallback. Esta combinacion asegura que incluso si el stream gRPC se interrumpe momentaneamente, el agente eventualmente recuperara su estado correcto.

## Protocolo Gossip

Gossip es un protocolo de comunicacion peer-to-peer donde cada nodo comparte informacion con un subconjunto aleatorio de sus vecinos en cada ciclo. La informacion se propaga gradualmente por todo el cluster como una epidemia controlada. No existe un punto central de comunicacion: cada nodo es simultaneamente emisor y receptor, y la convergencia ocurre porque cada nodo actualiza su vista del mundo con la informacion que recibe de sus pares.

Los orquestadores usan gossip para tres tipos de informacion: descubrimiento de nodos nuevos, deteccion de nodos caidos y propagacion de configuracion de red. Swarm utiliza el protocolo Serf (basado en SWIM) para gossip entre managers y workers. Nomad usa Serf para gossip entre agentes. Kubernetes, aunque no usa gossip nativamente, integra Consul (que implementa Serf) en许多 setups de produccion.

```
┌─────── Propagación Gossip (Ciclo 1) ────────┐
│                                               │
│  Node A sabe que Node C cayó                  │
│  Node A le avisa a Node B (aleatorio)         │
│                                               │
│  Node A ──► Node B                            │
│            "Node C caído"                      │
│                                               │
│  Node C ──(no responde)──► X                  │
└───────────────────────────────────────────────┘

┌─────── Propagación Gossip (Ciclo 2) ────────┐
│                                               │
│  Node B ahora sabe que Node C cayó            │
│  Node B le avisa a Node D (aleatorio)         │
│                                               │
│  Node B ──► Node D                            │
│            "Node C caído"                      │
│                                               │
│  Node A ──► Node E (otro aleatorio)           │
│            "Node C caído"                      │
│                                               │
└───────────────────────────────────────────────┘

┌─────── Convergencia (Ciclo N) ──────────────┐
│                                               │
│  Todos los nodos saben que Node C cayó        │
│  El control plane registra el fallo           │
│  El scheduler reasigna los workloads de C     │
│                                               │
└───────────────────────────────────────────────┘
```

```bash
# En Docker Swarm, gossip opera entre managers via Serf
# Ver los miembros del gossip ring
docker info | grep -A5 "Swarm"

# Inspeccionar el estado de gossip en Nomad
nomad member list

# Salida esperada:
# Name           Address       Port  Tags
# server-1       10.0.1.10     4648  build=1.6.0,dc=dc1,id=a1b2c3,role=server
# client-1       10.0.1.20     4648  build=1.6.0,dc=dc1,id=d4e5f6,role=client
# client-2       10.0.1.21     4648  build=1.6.0,dc=dc1,id=g7h8i9,role=client

# Ver los eventos de gossip en Nomad
nomad node status -verbose <node-id> | grep -A10 "Meta"
```

```hcl
# Configuracion de Serf/gossip en Nomad
bind_addr = "0.0.0.0"

# Puerto gossip (por defecto 4648)
addresses {
  rpc      = "10.0.1.10"
  serf     = "10.0.1.10"
  cluster  = "10.0.1.10"
}

# Parametros avanzados de Serf para optimizar la propagacion
# Tiempo maximo para que un nodo sea declarado muerto
server {
  # Heartbeat timeout: 10 segundos
  # Si un servidor no responde en 5 ciclos, se marca como fallido
  heartbeat_grace   = "10s"
  
  # El gossip entre servidores usa encriptacion
  encrypt = "clave-simetrica-en-base64"
}

# Configuracion de reintentos para nodos que se reconectan
client {
  # Maximo tiempo de espera para reconectar
  network_interface = "eth0"
  
  # Parametros de Serf para clientes
  meta {
    # Tags customizadas que se propagan via gossip
    "gpu"    = "true"
    "disk"   = "ssd"
    "region" = "eu-west-1"
  }
}
```

```bash
# Verificar la salud del gossip ring en Consul (integrado con Nomad)
consul members

# Ver el estado de los serf agents
nomad operator raft list-peers

# Generar una clave de encriptacion para gossip
nomad operator keygen
# Usar la clave generada en la config de todos los servidores
```

## Event-driven: reacciones automaticas

El modelo event-driven va un paso mas alla del pull y push: en lugar de que los componentes activen comunicacion, el sistema genera eventos automaticamente ante cambios de estado y los distribuye a los suscriptores interesados. Kubernetes implementa este patron con su sistema de informers: cada controller registra una escucha (watch) sobre el tipo de recurso que le interesa, y el API Server le notifica inmediatamente cuando ese recurso cambia.

Este modelo es la base del ciclo de control. Cuando un nodo muere, el API Server emite un evento `NodeNotReady`. El controller de nodos recibe este evento y marca los pods de ese nodo como candidates para reprogramacion. El controller de endpoints actualiza las tablas de servicio. El HPA (HorizontalPodAutoscaler) recibe metricas de los pods y ajusta el numero de replicas. Todo ocurre sin que ningun componente pregunte activamente por cambios.

```
┌─────── Event-driven en Kubernetes ────────┐
│                                           │
│  Evento: Node worker-3 NotReady          │
│      │                                    │
│      ├──► Controller de Nodos             │
│      │    Marca pods como "Terminating"   │
│      │                                    │
│      ├──► Controller de Endpoints         │
│      │    Elimina endpoints del nodo       │
│      │                                    │
│      ├──► Controller de ReplicaSet        │
│      │    Detecta déficit, crea nuevos    │
│      │                                    │
│      └──► HPA                            │
│           Evalúa si necesita escalar      │
│                                           │
│  Todos estos controllers reaccionan       │
│  AUTOMÁTICAMENTE al mismo evento          │
└───────────────────────────────────────────┘
```

```bash
# Ver eventos del cluster en tiempo real
kubectl get events -w

# Ver eventos especificos de un namespace
kubectl get events -n produccion --sort-by='.lastTimestamp'

# Ver el historial de eventos de un pod
kubectl describe pod <pod-name> | grep -A20 "Events"

# Filtrar eventos por tipo
kubectl get events --field-selector type=Warning
kubectl get events --field-selector reason=NodeNotReady

# En Nomad, ver las evaluaciones (equivalente a eventos)
nomad eval list
nomad eval status <eval-id>

# Ver las respuestas a eventos en Consul
consul watch -type=service -service=mi-servicio /scripts/handler.sh
```

```yaml
# Ejemplo de Kubernetes Event que muestra el ciclo completo
# Evento tipico de un pod que falla y se recupera automaticamente
apiVersion: v1
kind: Event
metadata:
  name: mi-pod.17abc123456def
  namespace: produccion
involvedObject:
  apiVersion: v1
  kind: Pod
  name: mi-pod
  namespace: produccion
reason: BackOff
message: >-
  Back-off restarting failed container
  El contenedor fallo 3 veces en los ultimos 5 minutos
  El kubelet esta esperando 300s antes de reintentar
type: Warning
firstTimestamp: "2026-07-25T10:30:00Z"
lastTimestamp: "2026-07-25T10:45:00Z"
count: 15
```

```yaml
# Ejemplo de Custom Resource Definition que genera eventos
# para un controller personalizado
apiVersion: events.k8s.io/v1
kind: Event
metadata:
  name: backup-completado.987654321
  namespace: kube-system
  labels:
    event-type: backup
    severity: info
reason: BackupCompleted
message: "Backup de etcd completado exitosamente. Tamano: 2.3GB"
involvedObject:
  apiVersion: apps/v1
  kind: Deployment
  name: etcd-backup-controller
  namespace: kube-system
reportingController: etcd-backup-controller
reportingInstance: controller-1
action: Backup
type: Normal
```

> El patron event-driven es lo que permite que un cluster de Kubernetes con 5000 nodos se autorrepare sin intervencion humana. Cada evento desencadena una cadena de reacciones en multiples controllers, cada uno especializado en un aspecto del estado del sistema. Esta composicion de reacciones es la clave de la escalabilidad operativa de los orquestadores modernos.

