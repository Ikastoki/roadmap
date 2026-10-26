# Arquitectura Cliente-Servidor

La arquitectura cliente-servidor es el patron dominante en orquestadores de contenedores modernos. La separacion entre quien toma decisiones (servidor) y quien ejecuta trabajo (cliente) permite escalar ambos lados independientemente, implementar mecanismos de seguridad granulares y mantener la disponibilidad del sistema incluso ante fallos parciales. Todos los orquestadores significativos — Kubernetes, Nomad, Docker Swarm — adoptan este patron con variaciones en el numero de componentes y los protocolos de comunicacion.

La comunicacion entre el cliente y el servidor nunca es directa en un entorno seguro. Cada peticion pasa por un canal cifrado con TLS, y en configuraciones de produccion se exige autenticacion mutua (mTLS) donde tanto el cliente como el servidor presentan certificados validos. Esto garantiza que ni un nodo comprometido pueda suplantar al control plane, ni un servidor falso pueda recibir instrucciones de los workers.

```
┌──────── Flujo completo de una operación ────────┐
│                                                  │
│  1. kubectl apply -f deployment.yaml             │
│     │                                            │
│     ▼                                            │
│  2. CLI ──HTTPS──► API Server (Control Plane)    │
│     │                                            │
│     ▼                                            │
│  3. API Server ──valida──► etcd (almacena)       │
│     │                                            │
│     ▼                                            │
│  4. Scheduler detecta pod pendiente              │
│     │                                            │
│     ▼                                            │
│  5. Scheduler ──asigna──► etcd (registra)        │
│     │                                            │
│     ▼                                            │
│  6. Kubelet del nodo detecta asignación          │
│     │                                            │
│     ▼                                            │
│  7. Kubelet ──ejecuta──► containerd ──► Pod      │
│     │                                            │
│     ▼                                            │
│  8. Kubelet reporta estado ──► API Server        │
│                                                  │
└──────────────────────────────────────────────────┘
```

## Flujo de una peticion

Cada operacion en el orquestador sigue un flujo predecible que involucra multiples componentes en secuencia. Analizar este flujo revela como la arquitectura cliente-servidor distribuye la responsabilidad y como el sistema tolera fallos en cada punto de la cadena.

Cuando un usuario ejecuta `kubectl apply -f deployment.yaml`, la CLI construye un objeto JSON a partir del YAML, lo firma con el certificado del usuario y lo envia al API Server via HTTPS. El API Server valida las credenciales, verifica que el usuario tiene permisos para crear el recurso solicitado en el namespace objetivo, valida la estructura del objeto y almacena la especificacion en etcd. En este punto, el estado deseado ha quedado registrado.

El scheduler, que monitorea etcd continuamente, detecta que hay pods sin asignar a un nodo. Evaluando los requisitos de recursos, restricciones de afinidad y politicas de tolerancia, selecciona el nodo optimo y escribe su decision de vuelta en etcd. El kubelet del nodo seleccionado, que tambien monitorea etcd o recibe notificaciones via el API Server, detecta que tiene un nuevo pod asignado y procede a crear los contenedores a traves del runtime.

```bash
# Flujo observable con kubectl en tiempo real
kubectl get events --sort-by='.lastTimestamp' -w

# Ejemplo de salida esperada:
# TYPE    REASON              OBJECT           MESSAGE
# Normal  Scheduled           pod/mi-app       Successfully assigned prod/mi-app to worker-3
# Normal  Pulling             pod/mi-app       Pulling image "nginx:1.25"
# Normal  Pulled              pod/mi-app       Successfully pulled image "nginx:1.25"
# Normal  Created             pod/mi-app       Created container nginx
# Normal  Started             pod/mi-app       Started container nginx
# Normal  Killing             pod/mi-app-ant   Stopping container nginx (actualización)
```

> Cada paso del flujo es auditable. `kubectl describe pod <nombre>` muestra la historia completa de planificacion, pull de imagenes, creacion de contenedores y health checks. Esta trazabilidad es esencial para diagnosticar problemas en clusters de produccion.

## Comunicacion segura con TLS/mTLS

La seguridad en la comunicacion entre componentes del orquestador se basa en certificados TLS. En una configuracion minima, el API Server presenta un certificado al cliente para que este pueda verificar que esta hablando con el servidor correcto (TLS unidireccional). En una configuracion de produccion, se exige mTLS: tanto el cliente como el servidor presentan certificados, validandose mutuamente en cada conexion.

Kubernetes genera y gestiona internamente un conjunto de certificados que se usan para la comunicacion entre componentes. El `kubeadm init` genera automaticamente la CA (Certificate Authority) del cluster, los certificados del API Server, los de etcd, y los de los kubelets. Cada kubelet recibe su propio certificado unico, firmado por la CA del cluster, lo que le permite autenticarse de forma segura ante el API Server.

```
┌─────── Infraestructura de Certificados ────────┐
│                                                  │
│  CA del Cluster                                  │
│  (kubeadm ca.crt / ca.key)                      │
│       │                                          │
│       ├──► Certificado API Server                │
│       │    (apiserver.crt / apiserver.key)       │
│       │    Uso: Serve TLS en puerto 6443         │
│       │                                          │
│       ├──► Certificado etcd                      │
│       │    (server.crt / server.key)             │
│       │    Uso: Serve TLS en puerto 2379         │
│       │                                          │
│       ├──► Certificado kubelet-1                 │
│       │    (kubelet-1.crt / kubelet-1.key)       │
│       │    Uso: Autenticarse ante API Server     │
│       │                                          │
│       └──► Certificado kubelet-2                 │
│            (kubelet-2.crt / kubelet-2.key)       │
│            Uso: Autenticarse ante API Server     │
│                                                  │
└──────────────────────────────────────────────────┘
```

```bash
# Ver los certificados del cluster
ls -la /etc/kubernetes/pki/

# Verificar la cadena de certificados del API Server
openssl s_client -connect api-server:6443 -showcerts </dev/null 2>/dev/null | \
  openssl x509 -noout -subject -issuer -dates

# Verificar que un kubelet se autentica correctamente
openssl s_client -connect worker-1:10250 \
  -cert /var/lib/kubelet/pki/kubelet-client-current.pem \
  -key /var/lib/kubelet/pki/kubelet-client-current.pem \
  </dev/null 2>/dev/null | openssl x509 -noout -subject

# Generar un certificado para un usuario manualmente
openssl genrsa -out usuario.key 2048
openssl req -new -key usuario.key -out usuario.csr \
  -subj "/CN=usuario/O=grupo-dev"
# Firmar con la CA del cluster
openssl x509 -req -in usuario.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out usuario.crt -days 365
```

> En Nomad, la autenticacion mTLS entre servidores y clientes se configura con los archivos `ca-cert`, `cert` y `key` tanto en el bloque `server` como en `client`. Los clientes solo confian en los certificados firmados por la misma CA, lo que impide que un servidor rogue pueda unirse al cluster.

## Alta disponibilidad del Control Plane

La alta disponibilidad del control plane se logra ejecutando multiples instancias de cada componente y distribuyendo el trafico entre ellas. En Kubernetes, esto implica multiples API Servers detras de un balanceador de carga, multiples instancias de etcd formando un cluster con consenso distribuido, y multiples instancias del scheduler y controller manager donde solo una de cada tipo es activa (lider) y las demas estan en standby.

El punto critico de disponibilidad es etcd. Dado que etcd almacena todo el estado del cluster, la perdida del cluster de etcd significa la perdida completa de la capacidad de orquestacion. Por esta razon, etcd se ejecuta tipicamente en nodos dedicados al control plane, con discos SSD y multiples replicas geograficas en configuraciones avanzadas.

```
┌─────── Control Plane con HA ─────────────────┐
│                                                │
│  ┌──── Load Balancer (VIP / DNS) ────────┐    │
│  │  10.0.0.100:6443                       │    │
│  └──┬──────────┬──────────┬──────────┬───┘    │
│     │          │          │          │         │
│  ┌──▼──┐   ┌──▼──┐   ┌──▼──┐   ┌──▼──┐     │
│  │ API │   │ API │   │ API │   │ API │       │
│  │ Srv │   │ Srv │   │ Srv │   │ Srv │       │
│  │  #1 │   │  #2 │   │  #3 │   │  #4 │       │
│  └──┬──┘   └──┬──┘   └──┬──┘   └──┬──┘     │
│     │          │          │          │         │
│     └──────────┴────┬─────┴──────────┘         │
│                     │                          │
│  ┌─────────── etcd Cluster (3/5 nodos) ─────┐ │
│  │                                           │ │
│  │  ┌──────┐   ┌──────┐   ┌──────┐         │ │
│  │  │etcd-1│──►│etcd-2│──►│etcd-3│         │ │
│  │  │Leader│   │Foll. │   │Foll. │         │ │
│  │  └──────┘   └──────┘   └──────┘         │ │
│  │                                           │ │
│  │  Consenso: 2/3 nodos para aceptar writes │ │
│  └───────────────────────────────────────────┘ │
│                                                │
└────────────────────────────────────────────────┘
```

```bash
# Verificar el estado del cluster de etcd
ETCDCTL_API=3 etcdctl member list -w table \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Salida esperada:
# +--------+---------+---------+---------------------------+---------------------------+
# |   ID   |  NAME   | PEERS   |        CLIENT URLS        |       PEER URLS           |
# +--------+---------+---------+---------------------------+---------------------------+
# | 8a39b..| etcd-1  |         | https://10.0.1.11:2379    | https://10.0.1.11:2380    |
# | b2c4d..| etcd-2  |         | https://10.0.1.12:2379    | https://10.0.1.12:2380    |
# | c3d5f..| etcd-3  |         | https://10.0.1.13:2379    | https://10.0.1.13:2380    |
# +--------+---------+---------+---------------------------+---------------------------+

# Verificar el lider actual de etcd
ETCDCTL_API=3 etcdctl endpoint status \
  --endpoints=https://10.0.1.11:2379,https://10.0.1.12:2379,https://10.0.1.13:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  -w table
```

```yaml
# Configuracion de kubeadm para cluster HA con stacked etcd
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v1.29.0
controlPlaneEndpoint: "api-cluster.prod.local:6443"
etcd:
  local:
    dataDir: /var/lib/etcd
    extraArgs:
      listen-metrics-urls: "http://0.0.0.0:2381"
apiServer:
  certSANs:
    - "api-cluster.prod.local"
    - "10.0.0.100"
    - "10.0.1.10"
    - "10.0.1.11"
    - "10.0.1.12"
    - "10.0.1.13"
networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
---
# Para unir un segundo nodo al control plane
apiVersion: kubeadm.k8s.io/v1beta3
kind: JoinConfiguration
controlPlane:
  controlPlaneEndpoint: "api-cluster.prod.local:6443"
  certificateKey: "<clave-de-certificados>"
discovery:
  bootstrapToken:
    apiServerEndpoint: "api-cluster.prod.local:6443"
    caCertHashes:
      - "sha256:abc123..."
```

> La configuracion con "stacked etcd" ejecuta etcd en los mismos nodos que el API Server, lo que simplifica la gestion pero crea un punto de falla compartido. La configuracion con "external etcd" ejecuta etcd en nodos separados, ofreciendo mayor aislamiento pero增加了 la complejidad operativa. Para entornos criticos, external etcd es la recomendacion.

