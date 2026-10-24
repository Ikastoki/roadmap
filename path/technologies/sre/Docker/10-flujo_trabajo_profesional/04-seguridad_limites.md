# Seguridad en produccion y limites de Docker

Ejecutar contenedores en produccion sin las medidas de seguridad adecuadas equivale a compartir el kernel del servidor con procesos que podrian estar comprometidos. Docker proporciona multiples mecanismos de defensa, pero su aplicacion requiere comprender tanto las capacidades del motor como sus limitaciones inherentes. La seguridad de los contenedores no es automatica: es el resultado de una configuracion deliberada y de la adopcion de practicas rigurosas en cada capa del stack.

## Imagenes base escaneadas

El punto de partida de cualquier contenedor seguro es la imagen base. Una imagen oficial de Alpine, Ubuntu o Debian recibe actualizaciones de seguridad periodicas, pero entre una actualizacion y otra pueden pasar semanas. Escanear la imagen en busca de vulnerabilidades conocidas debe ser un paso obligatorio antes de cualquier despliegue.

```bash
# Escaneo con Trivy
trivy image --severity HIGH,CRITICAL ubuntu:22.04

# Escaneo con Docker Scout
docker scout analyze myapp:1.2.3

# Escaneo con Snyk
snyk container test myapp:1.2.3 --severity-threshold=high
```

El escaneo debe integrarse en la pipeline de CI para bloquear el push de imagenes con vulnerabilidades criticas. Es recomendable mantener una lista de imagenes base aprobadas y actualizarlas semanalmente.

```dockerfile
# Preferir imagenes slim o distroless para reducir superficie de ataque
FROM node:20-slim AS build
FROM gcr.io/distroless/nodejs20-debian12 AS production
COPY --from=build /app /app
CMD ["/app/server.js"]
```

Las imagenes distroless contienen unicamente la aplicacion y sus dependencias de runtime, sin shell, sin gestor de paquetes y sin utilidades del sistema. Esto elimina vectores de ataque comunes, aunque dificulta la depuracion en produccion.

## Contenedores sin privilegios

Por defecto, los procesos dentro de un contenedor se ejecutan como root. Si un atacante compromete la aplicacion, obtiene acceso root dentro del contenedor. Aunque los namespaces limitan ese acceso al host, sigue siendo una posicion peligrosa. La solucion es ejecutar la aplicacion con un usuario no privilegiado.

```dockerfile
FROM node:20-slim
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
WORKDIR /app
COPY --chown=appuser:appgroup . .
USER appuser
CMD ["node", "server.js"]
```

```bash
# En tiempo de ejecucion
docker run --user 1000:1000 nginx
```

Kubernetes refuerza este patron con `PodSecurityContext`:

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
  containers:
    - name: myapp
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop: ["ALL"]
```

## Seccomp y AppArmor

El kernel de Linux expone cientos de llamadas al sistema (syscalls). Un contenedor tipico solo necesita una fraccion de ellas. Seccomp (Secure Computing Mode) permite definir un perfil que filtra las syscalls que un proceso puede invocar.

Docker aplica un perfil seccomp por defecto que bloquea aproximadamente 50 syscalls peligrosas. Para aplicaciones que requieren un control mas estricto, se puede definir un perfil personalizado:

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": ["accept", "accept4", "access", "arch_prctl", "brk",
                 "clock_gettime", "close", "connect", "dup", "dup2",
                 "epoll_create", "epoll_ctl", "epoll_wait", "exit",
                 "exit_group", "fchmod", "fchown", "fcntl", "fstat",
                 "fsync", "ftruncate", "futex", "getcwd", "getdents",
                 "getegid", "geteuid", "getgid", "getpeername",
                 "getpid", "getppid", "getsockname", "getsockopt",
                 "gettid", "getuid", "listen", "lseek", "lstat",
                 "mkdir", "mmap", "mprotect", "munmap", "nanosleep",
                 "newfstatat", "openat", "pipe2", "poll", "ppoll",
                 "pread64", "pwrite64", "read", "readlink", "readlinkat",
                 "recvfrom", "recvmsg", "rename", "rmdir", "rt_sigaction",
                 "rt_sigprocmask", "rt_sigreturn", "sched_yield",
                 "sendmmsg", "sendmsg", "sendto", "set_robust_list",
                 "set_tid_address", "setsockopt", "shutdown", "sigaltstack",
                 "socket", "stat", "statx", "sysinfo", "uname",
                 "unlink", "unlinkat", "wait4", "write", "writev"],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

```bash
docker run --security-opt seccomp=custom-profile.json myapp
```

AppArmor complementa a Seccomp controlando lo que un proceso puede hacer a nivel de sistema de archivos, red y capacidades. Docker proporciona un perfil por defecto (`docker-default`) que restringe el acceso a recursos sensibles.

```bash
# Cargar un perfil AppArmor personalizado
apparmor_parser -r -W myapp-profile

# Ejecutar el contenedor con el perfil
docker run --security-opt apparmor=myapp-profile myapp
```

## User namespaces

El user namespace permite mapear el usuario root dentro del contenedor a un usuario no privilegiado en el host. Esto significa que incluso si un atacante escapa del contenedor con uid 0, en el host sera el uid 65534 (nobody) u otro no privilegiado.

```json
// /etc/docker/daemon.json
{
  "userns-remap": "default"
}
```

Con `userns-remap` activado, Docker crea automaticamente un mapping que asigna el rango de UIDs del contenedor a un subrango no privilegiado en el host. Esto rompe la equivalencia uid 0 = root, anadiendo una capa de aislamiento adicional.

> Activar user namespaces tiene efectos secundarios: los volumenes bind mount pueden requerir ajustes de permisos, y algunas aplicaciones que dependen de capacidades privilegiadas dejaran de funcionar. Por esta razon, no es la opcion predeterminada en muchas distribuciones.

## Docker secrets y Vault

Gestionar secretos (contrasenas, tokens, claves API) en contenedores es un problema clasico. Pasarlos como variables de entorno es fragil: las variables de entorno son visibles en `docker inspect`, en logs y en procesos hijos.

Docker Swarm incluye un sistema de secretos nativo que cifra los datos en reposo y en transito, y los monta como archivos en `/run/secrets/`:

```bash
# Crear un secreto en Swarm
echo "mydbpassword" | docker secret create db_password -

# Usarlo en un servicio
docker service create \
  --name myapp \
  --secret db_password \
  myapp:latest
```

```python
# Lectura del secreto desde la aplicacion
with open('/run/secrets/db_password') as f:
    password = f.read().strip()
```

Para entornos fuera de Swarm o para una gestion mas sofisticada, HashiCorp Vault es el estandar. Vault ofrece rotacion automatica de secretos, politicas de acceso granular y auditoria.

```bash
# Autenticacion y obtencion de secreto via API
vault login -method=approle role_id=<id> secret_id=<secret>
vault kv get -mount=secret myapp/database
```

```go
// Integracion desde la aplicacion
import vault "github.com/hashicorp/vault/api"

client, _ := vault.NewClient(vault.DefaultConfig())
client.SetToken(vaultToken)
secret, _ := client.Logical().Read("secret/data/myapp/database")
password := secret.Data["data"].(map[string]interface{})["password"]
```

Los sidecars de Vault (como `vault-agent`) pueden inyectar secretos directamente en el sistema de archivos del contenedor, emulando el comportamiento de Docker Swarm secrets pero con las capacidades completas de Vault.

## Limites de Docker como plataforma

Docker resuelve brillantemente el problema del empaquetado y la ejecucion de aplicaciones, pero no es un orquestador a gran escala. Cuando el numero de servidores supera la decena, o cuando se requiere autoescalado, balanceo de carga avanzado, actualizaciones rolling complejas o politicas de afinidad, las carencias de Docker como plataforma autonoma se hacen evidentes.

| Caracteristica | Docker solo | Kubernetes |
|---|---|---|
| Autoescalado | No nativo | Horizontal Pod Autoscaler |
| Actualizaciones rolling | Limitado (Swarm) | Estrategias multiples |
| Balanceo de carga | Manual o Swarm | Services + Ingress |
| Almacenamiento persistente | Volumenes locales | CSI, PV/PVC |
| Politicas de red | Redes bridge/overlay | Network Policies |
| Service discovery | DNS de Compose/Swarm | KubeDNS / CoreDNS |
| Rollback | Manual | Automatizado |
| Multi-tenancy | No | Namespaces + RBAC |

La transicion natural desde Docker hacia Kubernetes no implica abandonar Docker como herramienta de desarrollo. Al contrario: Docker sigue siendo el formato de empaquetado universal. Las imagenes que se construyen con `docker build` son las mismas que se ejecutan en un cluster Kubernetes. El cambio esta en la capa de orquestacion, no en el formato de empaquetado ni en las practicas de construccion de imagenes.

```bash
# La misma imagen funciona en Docker y en Kubernetes
docker build -t myapp:latest .
docker run -p 3000:3000 myapp:latest

# Y tambien en Kubernetes
kubectl create deployment myapp --image=myapp:latest
kubectl expose deployment myapp --port=3000 --type=LoadBalancer
```

## Transicion natural a Kubernetes

Migrar de Docker a Kubernetes no requiere reescribir aplicaciones ni reconstruir imagenes. Los pasos tipicos son:

1. **Contenerizar la aplicacion** (ya hecho con Docker)
2. **Dividir en servicios** si la aplicacion es monolitica
3. **Externalizar configuracion**: variables de entorno, secretos, configmaps
4. **Anadir health checks** (liveness, readiness, startup)
5. **Definir recursos declarativos** en YAML de Kubernetes
6. **Adoptar Helm** para empaquetar y versionar los charts
7. **Integrar un operador de ingress** (nginx-ingress, Traefik, Istio)

Durante la transicion, herramientas como `kompose` convierten automaticamente archivos Compose a recursos de Kubernetes:

```bash
kompose convert -f compose.yaml
# Genera: deployment.yaml, service.yaml, etc.
```

> El resultado de kompose es un punto de partida, no un destino. La configuracion generada requiere ajustes manuales para aprovechar las capacidades nativas de Kubernetes.

Docker sigue siendo esencial en el flujo de trabajo moderno: el desarrollador construye y prueba con Docker, el CI pipeline escanea y firma las imagenes con Docker, y el orquestador (Kubernetes, Nomad, Swarm) las ejecuta en produccion. Docker no desaparece en la transicion a Kubernetes; se consolida como la capa de empaquetado universal sobre la que se construye toda la infraestructura.

