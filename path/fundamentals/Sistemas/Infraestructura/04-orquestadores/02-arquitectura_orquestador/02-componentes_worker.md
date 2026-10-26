# Componentes del Worker Node

Los nodos worker son las maquinas donde se ejecutan realmente las cargas de trabajo orquestadas. Mientras el control plane se encarga de decidir que ejecutar y donde, los workers ejecutan esas decisiones y reportan su estado de vuelta. Cada worker ejecuta tres componentes fundamentales: un agente que se comunica con el control plane, un runtime de contenedores que gestiona los ciclos de vida de las imagenes y contenedores, y un proxy de red que gestiona el enrutamiento local, el descubrimiento de servicios y las reglas de firewall.

La arquitectura de un worker esta disenada para ser lo mas ligera posible. El agente consume pocos recursos y actua como un cliente delegado: recibe instrucciones, las ejecuta y reporta. Si el agente falla, los contenedores ya en ejecucion continuan funcionando hasta que el control plane detecta la perdida de conectividad y reasigna las cargas. Esta resiliencia es intencional: un worker debe tolerar la perdida momentanea de conectividad con el control plane sin interrumpir el servicio.

```
┌──────────── Worker Node ─────────────┐
│                                       │
│  ┌─────────────────────────────────┐  │
│  │         Agente (Kubelet/        │  │
│  │         Nomad Client/Swarm)     │  │
│  │  ┌──────────┐  ┌────────────┐  │  │
│  │  │ Recibe   │  │ Reporta    │  │  │
│  │  │ instruc. │  │ estado     │  │  │
│  │  └────┬─────┘  └─────▲──────┘  │  │
│  └───────┼──────────────┼─────────┘  │
│          │              │             │
│          ▼              │             │
│  ┌───────────────┐      │             │
│  │  Container    │      │             │
│  │  Runtime      │──────┘             │
│  │  (containerd) │                    │
│  └───────┬───────┘                    │
│          │                            │
│  ┌───────▼───────┐  ┌──────────────┐  │
│  │  Contenedores │  │  Proxy Red   │  │
│  │  (workloads)  │  │  (iptables/  │  │
│  │               │  │   CNI/VPN)   │  │
│  └───────────────┘  └──────────────┘  │
│                                       │
└───────────────────────────────────────┘
```

## Agente del nodo

El agente es el representante local del control plane en cada worker. En Kubernetes se llama `kubelet`, en Nomad es el `nomad agent -client`, y en Docker Swarm es el `dockerd` con el modulo Swarm activado. Su responsabilidad principal es mantener los contenedores asignados en ejecucion y reportar metricas y estado al control plane periodicamente.

El agente no solo ejecuta contenedores. Tambien gestiona el montaje de volumenes, configura las interfaces de red segun las politicas del CNI (Container Network Interface), ejecuta health checks programados y gestiona los secrets de forma local. Cuando el control plane asigna un workload a un nodo, el agente recibe la especificacion completa y traduce esa definicion declarativa en llamadas al runtime de contenedores.

```json
// Configuracion de kubelet en un worker
{
  "apiVersion": "kubelet.config.k8s.io/v1beta1",
  "kind": "KubeletConfiguration",
  "authentication": {
    "anonymous": {
      "enabled": false
    },
    "webhook": {
      "enabled": true,
      "cacheTTL": "2m0s"
    }
  },
  "authorization": {
    "mode": "Webhook"
  },
  "clusterDNS": ["10.96.0.10"],
  "clusterDomain": "cluster.local",
  "cgroupDriver": "systemd",
  "containerRuntimeEndpoint": "unix:///var/run/containerd/containerd.sock",
  "failSwapOn": true,
  "healthzBindAddress": "0.0.0.0",
  "healthzPort": 10248,
  "nodeStatusUpdateFrequency": "10s",
  "protectKernelDefaults": true,
  "readOnlyPort": 0,
  "rotateCertificates": true,
  "serverTLSBootstrap": true
}
```

```hcl
# Configuracion de un agente Nomad en modo cliente
client {
  enabled = true
  
  network_interface = "eth0"
  
  # Configurar los dispositivos de cgroup para los contenedores
  cgroup_parent = "nomad"
  
  # Intervalo de heartbeats al servidor
  heartbeat {
    interval = "15s"
    threshold = "10"
  }
  
  # Resource manager para planificar asignaciones
  reserved {
    cpu    = 500
    memory = 256
    network {
      reserved_ports = "22,8300,8301,8302,8500,8600"
    }
  }
}

# Runtime de contenedores
plugin "docker" {
  config {
    allow_privileged = false
    volumes {
      enabled = true
    }
    gc {
      image_delay    = "3m"
      interval       = "1m"
      container      = "2m"
      dangling_image = "3m"
    }
  }
}
```

```bash
# Ver el estado del kubelet en un nodo
systemctl status kubelet
journalctl -u kubelet -f

# Ver las asignaciones en un nodo Nomad
nomad node status <node-id>
nomad alloc status <alloc-id>

# Ver el estado del agente Swarm en un nodo
docker node ls
docker node inspect self
```

> Un error comun en produccion es deshabilitar el kubelet o el agente Nomad sin considerar que el control plane seguira intentando asignar workloads a ese nodo. Si el agente no puede reportar su estado, el control plane lo marca como no disponible despues del timeout de heartbeat, pero los contenedores existentes pueden seguir ejecutandose como huerfanos si没有人 monitoriza su estado.

## Runtime de contenedores

El runtime de contenedores es el componente que realmente crea, ejecuta y destruye contenedores. No todos los orquestadores usan el mismo runtime, y la eleccion impacta directamente en el rendimiento, la compatibilidad y la seguridad del cluster.

Kubernetes originalmente usaba Docker directamente, pero desde la version 1.24 elimino la dependencia de Docker (dockershim) y ahora soporta cualquier runtime compatible con la CRI (Container Runtime Interface). Los runtimes mas utilizados son `containerd` (el runtime predeterminado de Docker, usado por la mayoria de distribuciones de Kubernetes), `CRI-O` (desarrollado especificamente para Kubernetes por Red Hat) y `gVisor` (sandbox de Google que añade una capa de aislamiento adicional).

```
┌─────── Capas del Runtime ──────────────┐
│                                         │
│  Kubelet / Agente                       │
│       │                                 │
│       ▼  CRI (gRPC)                    │
│  ┌─────────────────────┐               │
│  │  containerd / CRI-O │               │
│  │                     │               │
│  │  ┌──────────────┐   │               │
│  │  │  OCI Runtime  │   │               │
│  │  │  (runc)       │   │               │
│  │  └──────┬───────┘   │               │
│  │         │           │               │
│  │  ┌──────▼───────┐   │               │
│  │  │  Namespaces  │   │               │
│  │  │  cgroups     │   │               │
│  │  │  Capabilities│   │               │
│  │  └──────────────┘   │               │
│  └─────────────────────┘               │
│                                         │
└─────────────────────────────────────────┘
```

```bash
# Verificar que containerd esta corriendo
systemctl status containerd

# Listar contenedores gestionados por containerd
crictl ps
crictl images
crictl pods

# Inspeccionar un contenedor especifico
crictl inspect <container-id>

# Cambiar la imagen base de un pod usando crictl
crictl pull docker.io/library/nginx:1.25-alpine
```

```toml
# Configuracion de containerd
version = 2

[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
    [plugins."io.containerd.grpc.v1.cri".containerd]
      default_runtime_name = "runc"
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          runtime_type = "io.containerd.runc.v2"
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true
        # Runtime alternativo con gVisor para workloads de mayor aislamiento
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runsc]
          runtime_type = "io.containerd.runsc.v1"
    [plugins."io.containerd.grpc.v1.cri".cni]
      bin_dir = "/opt/cni/bin"
      conf_dir = "/etc/cni/net.d"
```

> containerd y CRI-O comparten el mismo conjunto de imagenes de OCI, pero difieren en su interfaz. containerd expone una API gRPC completa (la CRI es solo un subconjunto), mientras que CRI-O implementa exclusivamente la CRI. En la practica, la diferencia de rendimiento es marginal; la decision se basa mas en el ecosistema de la distribucion de Kubernetes que se utilice.

## Proxy de red

El proxy de red en cada worker gestiona el trafico de entrada y salida de los contenedores en ese nodo. En Kubernetes, este rol esta cubierto por `kube-proxy`, que configura reglas de iptables o IPVS para redirigir el trafico hacia los endpoints correctos. En Docker Swarm, el modulo `overlay` del kernel y los servicios de discovery gestionan estas funciones. En Nomad, la integracion con Consul Connect implementa un sidecar proxy (basado en Envoy) que maneja el trafico de servicio.

El proxy de red resuelve dos problemas fundamentales. Primero, el enrutamiento: los contenedores tienen IPs asignadas dinamicamente, pero los clientes necesitan un punto de contacto estable. Segundo, el descubrimiento: cuando un servicio depende de otro, el proxy permite encontrar automaticamente las instancias disponibles sin hardcodear direcciones.

```bash
# Ver las reglas de iptables que kube-proxy configura
sudo iptables -t nat -L KUBE-SERVICES -n
sudo iptables -t nat -L KUBE-SEP-XXXXXXXX -n

# Ver los servicios y sus endpoints actuales
kubectl get endpoints <service-name> -o wide

# En Docker Swarm, ver la red overlay
docker network ls
docker network inspect my-overlay

# En Nomad con Consul Connect, ver los proxies sidecar
nomad alloc status <alloc-id> | grep -A5 "Connect"
```

```yaml
# Configuracion de kube-proxy en modo IPVS (mejor rendimiento que iptables)
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  scheduler: "rr"
  syncPeriod: "30s"
  minSyncPeriod: "2s"
iptables:
  syncPeriod: "30s"
  minSyncPeriod: "2s"
metricsBindAddress: "0.0.0.0:10249"
conntrack:
  maxPerCore: 32768
  tcpCloseWaitTimeout: "1h0m0s"
  tcpEstablishedTimeout: "24h0m0s"
```

```yaml
# Configuracion de una network policy en Kubernetes
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: permitir-solo-interno
  namespace: produccion
spec:
  podSelector:
    matchLabels:
      app: api-server
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: frontend
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: base-datos
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - namespaceSelector:
            matchLabels:
              name: kube-system
      ports:
        - protocol: UDP
          port: 53
```

> kube-proxy soporta dos modos principales: iptables y IPVS. El modo iptables usa cadenas de reglas lineales (O(n) en numero de servicios), lo que degrada el rendimiento con miles de servicios. IPVS usa estructuras hash en el kernel (O(1)), soportando decenas de miles de servicios sin perdida de rendimiento. Para clusters de mas de 500 servicios, IPVS es la eleccion recomendada.

