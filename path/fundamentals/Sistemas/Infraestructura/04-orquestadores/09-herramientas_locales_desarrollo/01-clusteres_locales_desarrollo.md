# Clusteres locales de desarrollo

La distancia entre el entorno de desarrollo local y el de produccion ha sido historicamente una fuente inagotable de bugs del tipo "en mi maquina funciona". Los orquestadores de contenedores introducen una capa de complejidad adicional: no basta con que un contenedor se ejecute correctamente, sino que la orquestacion, el networking, el almacenamiento y las politicas de seguridad deben comportarse de manera consistente a lo largo de todo el ciclo de vida del software. Los clusteres locales de desarrollo resuelven este problema proporcionando entornos de Kubernetes (o sistemas equivalentes) que replican fielmente la topologia de produccion dentro de una estacion de trabajo, permitiendo a los desarrolladores validar sus aplicaciones con la misma API y los mismos recursos que encontrarán en el clúster real. Cada herramienta adopta una estrategia distinta para lograr esto: desde maquinas virtuales completas hasta contenedores anidados, pasando por distribuciones ultraligeras diseñadas para consumo minimo de recursos.

## Minikube

Minikube es la herramienta oficial del proyecto Kubernetes para ejecutar un cluster de un solo nodo en una maquina local. Funciona levantando una maquina virtual (o un contenedor) que alberga todos los componentes del plano de control y un nodo de trabajo, simulando un cluster completo. Internamente, Minikube gestiona el ciclo de vida del clúster mediante un driver que puede ser Docker, VirtualBox, KVM, Hyper-V o Podman, cada uno con implicaciones distintas en cuanto a rendimiento y compatibilidad.

### Instalacion

```bash
# Linux (binario directo)
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# macOS (Homebrew)
brew install minikube

# Windows (Chocolatey)
choco install minikube

# Verificar instalacion
minikube version
```

### Uso basico

```bash
# Iniciar un cluster con el driver Docker (recomendado)
minikube start --driver=docker

# Verificar estado del cluster
minikube status

# Abrir el dashboard de Kubernetes
minikube dashboard

# Activar addons comunes
minikube addons enable ingress
minikube addons enable metrics-server
minikube addons enable registry

# Listar addons disponibles
minikube addons list

# Detener el cluster (preserva estado)
minikube stop

# Eliminar el cluster completamente
minikube delete
```

### Configuracion avanzada

```bash
# Crear cluster con multiples nodos
minikube start --nodes=3 --driver=docker

# Asignar recursos especificos
minikube start --cpus=4 --memory=8192 --disk-size=40g

# Usar una version especifica de Kubernetes
minikube start --kubernetes-version=v1.29.0

# Montar un directorio local en el clúster
minikube mount ./mi-aplicacion:/mnt/data

# Obtener la IP del cluster
minikube ip

# Acceder al servicio en un navegador
minikube service mi-servicio --url
```

> Minikube con el driver Docker es la opcion más rapida para inicio, ya que reutiliza el daemon Docker ya instalado sin necesidad de un hipervisor adicional. Sin embargo, el anidamiento de contenedores (Docker-in-Docker o Docker-over-Docker) puede generar problemas de red en ciertas configuraciones.

## Kind (Kubernetes IN Docker)

Kind toma un enfoque radicalmente distinto: en lugar de levantar una maquina virtual, ejecuta cada nodo del cluster de Kubernetes como un contenedor Docker. Cada "nodo" es un contenedor que ejecuta los servicios systemd necesarios (kubelet, kube-proxy, contenedor runtime) y se comunica con el resto del cluster a traves de una red Docker dedicada. Esta arquitectura la hace extremadamente rapida para crear y destruir clusters, lo que la convierte en la herramienta preferida para pruebas automatizadas y pipelines de CI/CD.

### Instalacion

```bash
# Linux / macOS (binario)
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-$(uname)-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# macOS (Homebrew)
brew install kind

# Go install
go install sigs.k8s.io/kind@latest

# Verificar
kind version
```

### Uso basico

```bash
# Crear un cluster por defecto (un nodo)
kind create cluster

# Nombrar el cluster
kind create cluster --name mi-cluster

# Verificar nodos
kubectl get nodes

# Eliminar el cluster
kind delete cluster --name mi-cluster
```

### Clusteres multicopia con KindConfig

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
  - role: worker
  - role: worker
networking:
  disableDefaultCNI: false
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/16"
```

```bash
# Crear cluster con configuracion personalizada
kind create cluster --config kind-config.yaml --name produccion-local

# Listar clusters Kind
kind get clusters

# Cargar una imagen local en el cluster (evita pull de registry)
kind load docker-image mi-app:latest --name mi-cluster

# Exportar el kubeconfig
kind get kubeconfig --name mi-cluster > ~/.kube/kind-mi-cluster.yaml
```

> Kind es ideal para CI/CD porque los clusters se crean en segundos (típicamente 10-20s) y se destruyen instantáneamente. Muchas herramientas de testing de Kubernetes (como sonobuoy para certificaciones) usan Kind internamente.

## k3s y k3d

k3s es una distribucion de Kubernetes certificada por la CNCF, desarrollada originalmente por Rancher Labs, que elimina componentes pesados y los reemplaza por alternativas ligeras: SQLite en lugar de etcd como almacenamiento por defecto, containerd como runtime (sin Docker), y un binario unico que empaqueta todos los servicios del plano de control y del nodo de trabajo. k3d es un wrapper de k3s que ejecuta los nodos k3s dentro de contenedores Docker, proporcionando una experiencia similar a Kind pero con la distribucion k3s.

### Instalacion de k3s

```bash
# Instalacion rapida (script oficial)
curl -sfL https://get.k3s.io | sh -

# Verificar
sudo k3s kubectl get nodes

# El kubeconfig se genera en
sudo cat /etc/rancher/k3s/k3s.yaml

# Desinstalar
/usr/local/bin/k3s-uninstall.sh
```

### Instalacion y uso de k3d

```bash
# Instalar k3d
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash

# Crear un cluster con un nodo
k3d cluster create mi-cluster

# Crear cluster con multiples workers y puertos expuestos
k3d cluster create produccion \
  --servers 1 \
  --agents 3 \
  --port "8080:80@loadbalancer" \
  --port "8443:443@loadbalancer" \
  --volume "/data:/var/lib/rancher/k3s/storage@all" \
  --k3s-arg "--disable=traefik@server:0"

# Listar clusteres
k3d cluster list

# Exportar kubeconfig
k3d kubeconfig get mi-cluster

# Eliminar cluster
k3d cluster delete mi-cluster

# Añadir un nodo worker al cluster existente
k3d node create worker-nuevo --cluster mi-cluster --role agent --replicas 2
```

### Comparativa k3s vs k3d vs Kind

| Caracteristica | k3s | k3d | Kind |
|---|---|---|---|
| Binario unico | Si | No (usa Docker) | No (usa Docker) |
| Runtime por defecto | containerd | containerd | containerd |
| Almacenamiento | SQLite / etcd | SQLite / etcd | etcd |
| Arranque | ~5s | ~15s | ~15s |
| Uso de RAM | ~512MB | ~1GB | ~1GB |
| Entorno ideal | Edge / IoT / produccion | Desarrollo local | CI/CD / testing |

> k3s es la unica distribucion de esta lista pensada para produccion ademas de desarrollo. Se ejecuta nativamente en maquinas fisicas sin necesidad de Docker, lo que lo hace popular en edge computing, Raspberry Pi y clusters de baja potencia.

## Docker Desktop y Podman Desktop

Docker Desktop integra un cluster de Kubernetes activable desde la interfaz grafica o mediante CLI. Internamente, ejecuta un cluster de un solo nodo utilizando HyperKit (macOS), WSL2 (Windows) o un contenedor special (Linux). Podman Desktop ofrece funcionalidad similar pero basada en Podman y su sistema de "pods" con soporte para Kubernetes.

### Docker Desktop

```bash
# Activar Kubernetes via CLI (requiere Docker Desktop instalado)
# En macOS/Windows: desde la configuracion de Docker Desktop
# En Linux con plugin compose:
docker context use default

# Verificar que Kubernetes esta activo
kubectl cluster-info
kubectl get nodes

# El contexto de Docker Desktop se registra automaticamente
kubectl config get-contexts
```

> Docker Desktop proporciona un entorno unificado pero consume mas recursos que alternativas como k3d o Minikube. La version gratuita tiene restricciones de licencia para organizaciones con mas de 250 empleados o mas de 10 millones de dolares de facturacion anual.

### Podman Desktop

```bash
# Iniciar un cluster Kubernetes con Podman
# (requiere podman-desktop instalado y configurado)
podman machine init --cpus 2 --memory 4096 --disk-size 50
podman machine start

# Habilitar el soporte Kubernetes
podman machine set --kubernetes true

# Verificar el cluster
podman machine ssh -- kubectl get nodes
```

## Nomad en modo desarrollo

HashiCorp Nomad ofrece un modo de desarrollo local que ejecuta un agente en modo dev con un plano de control simplificado, almacenamiento en memoria y sin necesidad de un cluster de produccion. A diferencia de Kubernetes, Nomad usa un solo binario que maneja tanto el plano de control como el orquestado de tareas, y soporta multiples workloads: contenedores (Docker, Podman), binarios, Java, QEMU y mas.

### Instalacion

```bash
# Linux (binario)
curl -Lo nomad.zip https://releases.hashicorp.com/nomad/latest/nomad_linux_amd64.zip
unzip nomad.zip
sudo mv nomad /usr/local/bin/

# macOS (Homebrew)
brew install nomad

# Verificar
nomad version
```

### Uso basico

```bash
# Iniciar agente en modo dev (single-node, en memoria)
nomad agent -dev

# En otra terminal, verificar el status del cluster
nomad server members
nomad node status
```

> El modo `-dev` de Nomad es intencionalmente efimero: toda la informacion se pierde al detener el agente. Esto lo ideal para pruebas rapidas pero inadecuado para desarrollo iterativo donde se necesita preservar el estado entre sesiones.

### Ejemplo de job de prueba

```hcl
# ejemplo.nomad.hcl
job "web" {
  datacenters = ["dc1"]
  type = "service"

  group "frontend" {
    count = 1

    network {
      port "http" {
        static = 8080
      }
    }

    task "nginx" {
      driver = "docker"

      config {
        image = "nginx:alpine"
        ports = ["http"]
      }

      resources {
        cpu    = 100
        memory = 128
      }
    }
  }
}
```

```bash
# Ejecutar el job
nomad job run ejemplo.nomad.hcl

# Verificar el estado
nomad job status web

# Ver logs de la tarea
nomad alloc logs <ALLOC_ID>

# Detener el job
nomad job stop web
```

## Comparativa general

| Herramienta | Tipo de cluster | Multi-nodo | Tiempo de arranque | RAM minima | Ideales para |
|---|---|---|---|---|---|
| Minikube | VM / contenedor | Si | ~30s | 2GB | Desarrollo completo |
| Kind | Contenedores Docker | Si | ~15s | 1GB | CI/CD, testing |
| k3s | Binario nativo | Si | ~5s | 512MB | Edge, produccion |
| k3d | Contenedores + k3s | Si | ~15s | 1GB | Desarrollo rapido |
| Docker Desktop | Integrado | No | ~20s | 2GB | Desarrollo general |
| Nomad Dev | Agent unico | No | ~2s | 256MB | Multi-workload |

> La eleccion entre estas herramientas depende del contexto: para pipelines de CI/CD, Kind es la opcion estandar por su velocidad y facilidad de integracion. Para desarrollo diario con necesidades de networking complejo, Minikube o k3d proporcionan mayor control. Para entornos edge o con recursos limitados, k3s es insuperable.

