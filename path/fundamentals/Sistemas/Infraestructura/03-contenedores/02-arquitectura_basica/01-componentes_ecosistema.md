# Componentes del ecosistema de contenedores

El ecosistema de contenedores esta compuesto por multiples capas de software que trabajan juntas para construir, distribuir y ejecutar contenedores. Desde el runtime de bajo nivel que ejecuta el proceso aislado hasta la CLI que el desarrollador utiliza, cada componente tiene un rol especifico y bien definido.

## Arquitectura en capas

```
CLI (docker, podman)
    │
    ▼
API REST / Daemon
    │
    ▼
Container Engine (dockerd, containerd, CRI-O)
    │
    ▼
OCI Runtime (runc, crun, kata-containers)
    │
    ▼
Namespaces + Cgroups + UnionFS
    │
    ▼
Kernel de Linux
```

## Runtime de bajo nivel (OCI Runtime)

Es el componente que ejecuta el contenedor directamente. Recibe la configuracion del container engine y crea el proceso aislado con namespaces, cgroups y sistema de archivos en capas.

- **runc:** el runtime de referencia de la OCI. Creado por Docker y donado a la comunidad.
- **crun:** alternativa escrita en C, mas rapida y ligera que runc. Default en Podman desde 2021.
- **kata-containers:** ejecuta cada contenedor en una micro-VM para mayor aislamiento.

```bash
# runc: ejecutar contenedor directamente
# (normalmente no se usa directamente; el engine lo hace)
runc run mi-contenedor
```

## Container Engine / Daemon

Es la capa que gestiona el ciclo de vida de imagenes y contenedores: construccion, descarga, ejecucion, eliminacion, redes, volumenes. Se comunica con el runtime de bajo nivel y proporciona la API que usa la CLI.

- **Docker Engine (dockerd):** el motor mas popular. Arquitectura cliente-servidor con daemon centralizado.
- **containerd:** daemon de contenedores que Docker dono a CNCF. Es el runtime base de Kubernetes.
- **CRI-O:** daemon ligero disenado exclusivamente para Kubernetes (interfaz CRI).
- **Podman:** motor daemonless. No hay demonio central; cada operacion ejecuta un proceso independiente.

```bash
# Ver el daemon de Docker corriendo
ps aux | grep dockerd
# root  1234  dockerd --containerd /run/containerd/containerd.sock

# Ver containerd
ps aux | grep containerd
# root  1235  containerd
```

## CLI (Interfaz de comandos)

La herramienta que el desarrollador usa para interactuar con el engine. Se comunica con el daemon a traves de la API REST.

```bash
# Docker CLI
docker run -d nginx          # Ejecutar contenedor
docker build -t mi-app .     # Construir imagen
docker ps                     # Listar contenedores

# Podman CLI (compatible con Docker)
podman run -d nginx
podman build -t mi-app .
podman ps
```

## Registry (Registro)

Servicio que almacena y distribuye imagenes. Puede ser publico (Docker Hub) o privado (Harbor, AWS ECR).

```bash
# Docker Hub: registro publico por defecto
docker pull nginx:alpine     # Descargar desde Docker Hub
docker push usuario/mi-app   # Subir a Docker Hub

# Registro privado
docker pull registry.empresa.com/mi-app:1.0
```

## Composicion del ecosistema

| Componente | Ejemplo | Funcion |
|---|---|---|
| CLI | docker, podman | Interfaz de usuario |
| Engine | dockerd, containerd | Gestion de ciclo de vida |
| Runtime | runc, crun | Ejecucion de contenedores |
| Registry | Docker Hub, Harbor | Almacenamiento de imagenes |
| Compose | docker compose | Orquestacion local |
| Swarm/K8s | Docker Swarm, Kubernetes | Orquestacion distribuida |

> El ecosistema de contenedores esta disenado en capas claramente separadas. Un desarrollador puede usar Docker sin conocer runc; un administrador de Kubernetes puede cambiar containerd por CRI-O sin afectar las aplicaciones. Esta modularidad es lo que hace al ecosistema robusto y flexible.

