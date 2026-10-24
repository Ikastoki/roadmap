# Minikube

Desarrollar para Kubernetes sin un cluster disponible es como escribir codigo sin compilador: posible pero impractico. Minikube resuelve este problema ofreciendo un cluster de Kubernetes de un solo nodo que se ejecuta localmente, ya sea dentro de una maquina virtual o directamente sobre Docker. Su proposito no es simular un entorno de produccion, sino proporcionar un playground funcional donde desarrollar, depurar y probar aplicaciones antes de enviarlas a un cluster real. A diferencia de las soluciones cloud, Minikube arranca en segundos, no requiere credenciales de proveedor ni genera costos, y expone todas las APIs de Kubernetes para que el ciclo de desarrollo sea identico al de produccion.

## Instalacion

Minikube se distribuye como un unico binario estatico. La forma mas directa de instalarlo en Linux es descargarlo directamente desde su repositorio oficial:

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

En macOS, Homebrew ofrece una formula actualizada:

```bash
brew install minikube
```

En Windows, se puede instalar via Chocolatey, winget o descargando el ejecutable directamente desde GitHub Releases. Independientemente del sistema operativo, Minikube requiere un driver que proporcione el hipervisor subyacente. Las opciones mas comunes son:

- **Docker**: el driver por defecto, no requiere instalacion adicional si ya se tiene Docker.
- **VirtualBox**: multiplataforma, funcional pero mas pesado que Docker.
- **KVM**: nativo de Linux, mejor rendimiento que VirtualBox.
- **Hyper-V**: en Windows, requiere tenerlo habilitado.
- **Podman**: alternativa a Docker, soportada desde versiones recientes.

```bash
# Verificar la instalacion y el estado de los drivers disponibles
minikube version
minikube config --help
```

> Minikube no incluye Docker por defecto cuando se usa el driver Docker. En ese modo, comparte el daemon de Docker del host, lo que significa que las imagenes construidas localmente estan disponibles inmediatamente para el cluster sin necesidad de un registro de contenedores intermedio.

## Arranque y gestion del cluster

El comando fundamental es `minikube start`, que orquesta la creacion de la maquina virtual o el contenedor, la instalacion de los componentes de Kubernetes (API Server, kubelet, kube-proxy, CoreDNS, etc.), y la configuracion del archivo `~/.kube/config` para que kubectl pueda comunicarse con el nuevo cluster.

```bash
# Arrancar con la configuracion por defecto
minikube start

# Arrancar con recursos personalizados
minikube start --cpus=4 --memory=8192 --disk-size=20g

# Arrancar con un driver especifico
minikube start --driver=kvm2

# Arrancar una version especifica de Kubernetes
minikube start --kubernetes-version=v1.28.3
```

Minikube descarga la imagen del sistema operativo ligero (`minikube-image`) que contiene todos los binarios de Kubernetes. Durante el arranque, ejecuta `kubeadm init` internamente para inicializar el plano de control y configura el kubelet para que gestione los pods del sistema (etcd, scheduler, controller-manager).

Los comandos de gestion son intuitivos:

```bash
# Detener el cluster sin eliminar los datos
minikube stop

# Eliminar el cluster por completo
minikube delete

# Ver el estado del cluster
minikube status

# Pausar/reanudar todos los nodos (util para ahorrar recursos)
minikube pause
minikube unpause
```

## Dashboard web

Minikube incluye el dashboard oficial de Kubernetes, una interfaz web que permite inspeccionar el estado del cluster, visualizar pods, servicios, deployments, y ejecutar operaciones basicas sin usar kubectl.

```bash
# Abrir el dashboard en el navegador por defecto
minikube dashboard

# Obtener la URL sin abrir el navegador
minikube dashboard --url
```

El dashboard se despliega como un pod en el namespace `kubernetes-dashboard` y se expone mediante un proxy que Minikube gestiona automaticamente. Cuando se ejecuta `minikube dashboard`, se inicia un proceso en segundo plano que redirige el trafico local al dashboard del cluster.

## Addons

Minikube incorpora un sistema de addons que permite activar funcionalidades extra del ecosistema Kubernetes con un solo comando. Estos addons son componentes que normalmente requeririan instalacion manual, como ingress controllers, registries de imagenes, o herramientas de observabilidad.

```bash
# Listar todos los addons disponibles y su estado
minikube addons list

# Activar un addon
minikube addons enable ingress
minikube addons enable metrics-server
minikube addons enable registry

# Desactivar un addon
minikube addons disable ingress
```

Algunos de los addons mas utilizados:

- **ingress**: despliega el NGINX Ingress Controller, permitiendo probar reglas de enrutamiento HTTP localmente.
- **metrics-server**: necesario para el funcionamiento de Horizontal Pod Autoscaler (HPA) y `kubectl top`.
- **registry**: un registro privado de imagenes Docker accesible desde el cluster, util para desarrollo sin necesidad de Docker Hub.
- **dashboard**: el dashboard web de Kubernetes (suele activarse por defecto).
- **storage-provisioner**: aprovisiona automaticamente PersistentVolumes usando el almacenamiento del host.

> Los addons no son componentes de Kubernetes en si mismos, sino manifiestos YAML que Minikube aplica contra el cluster. Se puede inspeccionar el contenido de cada addon en el directorio `~/.minikube/addons/`.

## Interaccion con el cluster

Una vez que Minikube esta corriendo, toda la interaccion se realiza con kubectl, como si se tratara de un cluster remoto. Minikube simplemente configura el contexto de kubectl para que apunte a `https://192.168.x.x:8443` (en el caso del driver Docker, a `https://127.0.0.1:puerto_aleatorio`).

```bash
# Verificar que estamos en el contexto de Minikube
kubectl config current-context
# minikube

# Operaciones habituales
kubectl get nodes
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
minikube service nginx --url
```

El comando `minikube service` es especialmente util: expone un servicio del cluster en una URL accesible desde el host, gestionando automaticamente el tunel necesario para alcanzar el servicio desde fuera de la maquina virtual o contenedor.

## Limitaciones

Minikube es una herramienta de desarrollo, no de produccion. Sus limitaciones principales son:

- **Un solo nodo**: no permite probar comportamientos que dependen de la topologia del cluster, como tolerancias a fallos de nodo o scheduling entre multiples nodos.
- **Recursos limitados**: al compartir los recursos del host, no es adecuado para pruebas de carga o escalado real.
- **Alta disponibilidad**: no soporta multiples replicas del plano de control.
- **Persistencia**: al eliminar el cluster con `minikube delete`, se pierden todos los datos, incluyendo imagenes y almacenamiento persistente.

Para escenarios que requieren multiples nodos o mayor fidelidad con produccion, alternativas como Kind o clusters gestionados en cloud son mas apropiadas.

