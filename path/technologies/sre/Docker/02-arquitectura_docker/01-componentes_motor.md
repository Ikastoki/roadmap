# Componentes del motor de Docker

El motor de Docker no es un unico binario monolotico, sino un conjunto de componentes especializados que colaboran para construir, ejecutar y gestionar contenedores. Esta arquitectura modular, resultado de la progresiva separacion de responsabilidades, permite que cada componente evolucione de forma independiente y que otros proyectos del ecosistema (como Kubernetes) puedan utilizar componentes individuales sin depender de Docker en su totalidad. Comprender como encajan estas piezas es esencial para diagnosticar problemas, optimizar el rendimiento y disenar arquitecturas robustas.

## El demonio dockerd

El componente central del motor de Docker es `dockerd`, el demonio que se ejecuta en segundo plano y expone una API REST para gestionar contenedores, imagenes, redes y volumenes. `dockerd` es el punto de entrada principal y actua como orquestador de los demas componentes.

```bash
# Ver el estado del demonio Docker
sudo systemctl status docker

# Ver los logs del demonio
sudo journalctl -u docker -f

# La API REST de Docker escucha por defecto en /var/run/docker.sock
curl --unix-socket /var/run/docker.sock http://localhost/version
```

El demonio se configura a traves del archivo `/etc/docker/daemon.json` y acepta opciones de linea de comandos. Las configuraciones tipicas incluyen el driver de almacenamiento, los limites de recursos globales y las politicas de seguridad.

```json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
```

## Docker CLI: el cliente

La interfaz de linea de comandos de Docker, `docker`, es el cliente que se comunica con el demonio `dockerd` a traves de la API REST. El cliente puede ejecutarse en la misma maquina que el demonio o en una maquina remota, conectandose a traves de HTTPS o SSH.

```bash
# Comunicacion local a traves del socket Unix
docker ps

# Comunicacion remota a traves de TCP
docker -H tcp://192.168.1.100:2375 ps

# Comunicacion remota segura a traves de SSH
docker -H ssh://usuario@servidor ps
```

El cliente es un binario independiente que no requiere ejecutarse como root. Todo el trabajo pesado lo realiza el demonio. Esta separacion permite que el cliente y el demonio sean versiones diferentes, aunque se recomienda mantener compatibilidad.

```bash
# Ver versiones del cliente y del servidor
docker version
# Client: 24.0.7
# Server: 24.0.7
```

## containerd: el runtime de contenedores

`containerd` es el runtime de contenedores de alto nivel que Docker utiliza internamente desde la version 1.11. Es un componente separado que gestiona el ciclo de vida completo de los contenedores: creacion, ejecucion, detencion y eliminacion. `containerd` se comunica con `dockerd` a traves del protocolo gRPC.

```bash
# Ver el proceso containerd
ps aux | grep containerd

# containerd tiene su propia CLI para depuracion
sudo ctr images list
sudo ctr containers list
```

`containerd` se ha convertido en un componente critico del ecosistema, ya que es el runtime de contenedores que utiliza Kubernetes a traves de CRI (Container Runtime Interface). Docker, Kubernetes y otros orquestadores pueden compartir el mismo `containerd` subyacente.

```bash
# Verificar que containerd esta corriendo
sudo systemctl status containerd
# containerd.service - containerd container runtime
```

## runc: el runtime de bajo nivel

`runc` es el runtime de contenedores de bajo nivel, una implementacion de referencia de la especificacion OCI (Open Container Initiative). `runc` es el encargado de crear los namespaces y cgroups, ejecutar el proceso del contenedor y gestionar su ciclo de vida a nivel de sistema operativo.

```bash
# runc es el que realmente crea el contenedor a nivel de kernel
# Se puede usar directamente (aunque no es lo habitual)
sudo runc run contenedor
```

`containerd` utiliza `runc` (u otro runtime compatible con OCI como `crun`) para ejecutar los contenedores. La cadena completa es: `docker CLI -> dockerd -> containerd -> runc -> contenedor`.

## BuildKit: el constructor de imagenes

BuildKit es el motor de construccion de imagenes introducido en Docker 18.09. Reemplaza al builder legacy con un diseno moderno que ofrece construcciones paralelas, cache eficiente, montajes de cache y soporte para secretos.

```bash
# BuildKit se invoca automaticamente al construir
docker build -t mi-app .

# Para ver el progreso detallado
docker build --progress=plain -t mi-app .

# Construir con salida de depuracion
docker build --progress=plain --no-cache -t mi-app .
```

BuildKit procesa el Dockerfile como un grafo de dependencias, no como una secuencia lineal. Esto permite ejecutar en paralelo instrucciones independientes, como la descarga de multiples paquetes.

```dockerfile
# BuildKit puede ejecutar estas dos RUN en paralelo si son independientes
RUN apt-get update && apt-get install -y curl wget
RUN pip install -r requirements.txt
```

## Docker Compose: el orquestador local

Docker Compose (version 2, integrada como plugin de Docker) permite definir y ejecutar aplicaciones multicontenedor. Aunque originalmente era un binario separado, ahora forma parte del ecosistema Docker como un plugin gestionado por `docker compose`.

```bash
# Docker Compose como plugin
docker compose version
# Docker Compose version v2.24.2

# Comandos principales
docker compose up -d
docker compose down
docker compose logs -f
docker compose ps
```

Internamente, Docker Compose traduce la definicion YAML en llamadas a la API de Docker, creando redes, volumenes y contenedores segun la configuracion declarativa.

## Docker Networking: la capa de red

La gestion de red en Docker se implementa a traves del `libnetwork`, un modulo que proporciona abstracciones para diferentes tipos de redes. `libnetwork` implementa el modelo CNM (Container Network Model), que define conceptos como sandbox (entorno de red aislado), endpoint (punto de conexion) y network (grupo de endpoints que pueden comunicarse).

```bash
# Ver los drivers de red disponibles
docker info | grep -A 4 "Plugins"
# Plugins:
#  Volume: local
#  Network: bridge host ipvlan macvlan null overlay

# Ver las redes existentes
docker network ls
```

Los drivers de red se implementan como plugins que `libnetwork` carga dinamicamente. El driver `bridge` crea puentes de red Linux, `overlay` utiliza VXLAN para redes multi-host, y `macvlan` asigna direcciones MAC fisicas a los contenedores.

## Docker Volumes: la capa de almacenamiento

La gestion de volumenes se realiza a traves del driver de volumenes de Docker. El driver por defecto es `local`, que gestiona volumenes en el sistema de archivos del anfitrion. Docker tambien soporta drivers de terceros para almacenamiento en red como NFS, Ceph o Amazon EBS.

```bash
# Drivers de volumen disponibles
docker info | grep -A 4 "Plugins"
# Volume: local

# Crear un volumen con el driver local
docker volume create --driver local mi-volumen

# Los volumenes se almacenan en /var/lib/docker/volumes/
ls -la /var/lib/docker/volumes/
```

Cada volumen se implementa como un directorio en `/var/lib/docker/volumes/` que se monta en el sistema de archivos del contenedor. Cuando se utiliza un driver de terceros, el directorio se monta a traves de FUSE o un sistema de archivos en red.

## La interaccion entre componentes

El flujo completo de ejecucion de un contenedor ilustra como colaboran todos los componentes:

```bash
# 1. El usuario ejecuta:
docker run -d --name web -p 8080:80 nginx:latest

# 2. Docker CLI envia la peticion a dockerd via API REST
# 3. dockerd verifica la imagen localmente; si no existe, la descarga
#    consultando al registro (Docker Hub) via HTTPS
# 4. dockerd solicita a containerd que cree el contenedor
# 5. containerd configura las capas del sistema de archivos (overlay2)
# 6. containerd solicita a runc que ejecute el contenedor
# 7. runc crea namespaces, configura cgroups y ejecuta el proceso
# 8. El contenedor nginx se inicia y escucha en el puerto 80
# 9. dockerd configura el mapeo de puertos (-p 8080:80) via iptables
```

```bash
# Verificar la cadena de componentes
docker inspect web --format '{{.State.Pid}}'
# Con el PID, se puede ver el cgroup y los namespaces
cat /proc/$(docker inspect web --format '{{.State.Pid}}')/cgroup
ls -la /proc/$(docker inspect web --format '{{.State.Pid}}')/ns/
```

> La arquitectura modular de Docker no es accidental. Responde a la necesidad de separar responsabilidades y permitir la interoperabilidad. `containerd` puede ser utilizado directamente por Kubernetes sin pasar por `dockerd`. `runc` puede ser reemplazado por `crun` (escrito en C, mas rapido) sin cambios en el resto del sistema. Esta separacion de preocupaciones es lo que ha permitido que el ecosistema de contenedores evolucione tan rapidamente.

