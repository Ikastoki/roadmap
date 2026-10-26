# Arquitectura cliente-servidor vs. daemonless

La forma en que un motor de contenedores se comunica con el kernel y gestiona los contenedores varia significativamente entre motores. Docker utiliza un modelo cliente-servidor con un daemon centralizado; Podman utiliza un modelo daemonless donde cada operacion es un proceso independiente. Ambos enfoques tienen ventajas y desventajas que afectan la seguridad, la escalabilidad y la experiencia del usuario.

## Modelo cliente-servidor (Docker)

```
CLI (docker)
    │
    │ API REST (socket /var/run/docker.sock)
    ▼
Docker Daemon (dockerd)
    │
    ├── containerd (daemon de contenedores)
    │       │
    │       ▼
    │   runc (runtime)
    │       │
    │       ▼
    │   Kernel de Linux (namespaces, cgroups)
    │
    └── Networking (libnetwork, iptables)
```

**Caracteristicas:**
- Un daemon centralizado (dockerd) gestiona todo.
- La CLI se comunica con el daemon a traves de un socket Unix o TCP.
- El daemon gestiona imagenes, contenedores, redes, volumenes.
- Si el daemon falla, todos los contenedores siguen ejecutandose pero no se pueden gestionar nuevas operaciones.

**Ventajas:**
- Gestion centralizada y coherente.
- API REST estandar para integracion con herramientas.
- Estado persistente del daemon.

**Desventajas:**
- Single point of failure: si el daemon cae, no se pueden crear nuevos contenedores.
- El daemon necesita privilegios de root (por defecto).
- Un solo daemon gestiona todos los contenedores (posible bottleneck).

## Modelo daemonless (Podman)

```
CLI (podman)
    │
    ├── Ejecucion 1: podman run ──► crun ──► Kernel
    ├── Ejecucion 2: podman run ──► crun ──► Kernel
    └── Ejecucion 3: podman build ──► buildah ──► Kernel
```

**Caracteristicas:**
- No hay daemon centralizado.
- Cada operacion es un proceso independiente que termina cuando la operacion completa.
- Sin socket compartido, sin demonio que mantener.
- Ejecucion sin privilegios de root (rootless) por defecto.

**Ventajas:**
- Sin single point of failure.
- Seguridad mejorada: cada operacion es independiente.
- Sin daemon consuming recursos continuamente.
- Ejecucion rootless por defecto.

**Desventajas:**
- Sin estado persistente entre operaciones.
- Algunas funcionalidades menos maduras que Docker.

## Docker vs. Podman

| Aspecto | Docker | Podman |
|---|---|---|
| Daemon | Si (dockerd) | No (daemonless) |
| Privilegios | Root (por defecto) | Rootless (por defecto) |
| Socket | /var/run/docker.sock | No aplica |
| CLI | docker | podman (compatible) |
| Compatibilidad OCI | Si | Si |
| Gestión de pods | No nativo | Si (podman pod) |
| Seguridad | Menor (daemon con root) | Mayor (sin daemon, rootless) |

```bash
# Docker: requiere daemon corriendo
sudo systemctl start docker
docker run -d nginx

# Podman: sin daemon, sin sudo
podman run -d nginx
# Funciona sin privilegios de root
```

## Compatibilidad y transicion

Docker y Podman son compatibles a nivel de CLI y formato de imagenes. Un contenedor construido con Docker se ejecuta con Podman y viceversa.

```bash
# Alias para usar Podman como Docker
alias docker=podman

# O configurar el entorno
export DOCKER_HOST=unix:///run/user/1000/podman/podman.sock
```

> La eleccion entre daemon y daemonless es fundamentalmente una decision de arquitectura. Docker prioriza la compatibilidad y la experiencia del usuario; Podman prioriza la seguridad y la simplicidad. En entornos de produccion, ambos son validos; en entornos con requisitos de seguridad estrictos, la arquitectura daemonless de Podman tiene ventajas significativas.

