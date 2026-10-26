# Historia y evolucion de la contenerizacion

La contenerizacion no nacio con Docker en 2013. Sus conceptos raiz se remontan a las decadas de 1970 y 1980, cuando los sistemas operativos ya buscaban formas de aislar procesos sin duplicar hardware. La evolucion desde `chroot` hasta el ecosistema cloud-native actual refleja una busqueda constante de eficiencia, portabilidad y estandarizacion.

## Origenes: chroot (1979)

El antecesor mas antiguo de los contenedores es `chroot`, introducido en Unix V7 (1979). Cambia la raiz percibida de un proceso, aislándolo del resto del sistema de archivos. No es un contenedor moderno (no aísla red, usuarios ni procesos), pero establecio el principio de aislar un proceso del sistema host.

```bash
# chroot: cambiar la raiz del sistema
sudo chroot /nueva/raiz /bin/bash
# El proceso solo ve /nueva/raiz como /
# No puede acceder al resto del sistema de archivos
```

## FreeBSD Jails (2000)

FreeBSD introdujo las *jails*, que extendieron `chroot` con aislamiento de red, usuarios y procesos. Cada jail tiene su propia direccion IP, su propio sistema de archivos y sus propios usuarios, pero comparte el kernel de FreeBSD.

```
FreeBSD Host
├── Jail 1 (web server, 192.168.1.10)
├── Jail 2 (database, 192.168.1.11)
└── Jail 3 (mail server, 192.168.1.12)
```

## Solaris Zones (2004)

Sun Microsystems implemento *Zones* en Solaris 10, que permitian crear multiples entornos aislados en un solo servidor. Cada zone tenia su propio entorno de ejecucion pero compartia el kernel de Solaris.

## Linux Containers - LXC (2008)

LXC fue el primer motor de contenedores completo para Linux, combinando namespaces y cgroups para crear entornos aislados que se comportaban como VMs ligeras pero sin la sobrecarga de un SO invitado.

```bash
# LXC: crear un contenedor Ubuntu
sudo lxc-create -t download -n mi-contenedor -- -d ubuntu -r jammy -a amd64
sudo lxc-start -n mi-contenedor
sudo lxc-attach -n mi-contenedor
# Estas dentro de un entorno Ubuntu aislado, pero usando el kernel del host
```

## Docker (2013)

Docker no invento la contenerizacion; la hizo accesible. Con una interfaz de usuario simple, un ecosistema de imagenes y un registro centralizado (Docker Hub), Docker convirtio a los contenedores en la herramienta de despliegue de software mas utilizada del mundo.

```
Evolucion de la contenerizacion:
1979: chroot (aislamiento basico de archivos)
2000: FreeBSD Jails (aislamiento + red + usuarios)
2004: Solaris Zones (virtualizacion a nivel de SO)
2008: LXC (contenedores completos en Linux)
2013: Docker (estandarizacion y ecosistema)
2015: OCI (Open Container Initiative) - estandar de imagenes y runtime
2016: containerd (runtime estandar para Kubernetes)
2020: Podman (alternativa daemonless y rootless)
2024: Ecosistema maduro (Kubernetes, Helm, Istio, etc.)
```

## OCI: Open Container Initiative (2015)

Fundada por Docker, CoreOS y otros, la OCI establecio estandares para formato de imagenes y runtime de contenedores. Esto garantiza que las imagenes construidas con Docker se ejecuten en cualquier motor compatible (Podman, containerd, CRI-O).

## Ecosistema actual

El ecosistema de contenedores ha madurado significativamente:
- **Motores:** Docker, Podman, containerd, CRI-O.
- **Orquestadores:** Kubernetes, Docker Swarm, Nomad.
- **Seguridad:** Falco, Trivy, Cosign.
- **Redes:** Calico, Cilium, Flannel.
- **Almacenamiento:** Rook, Longhorn, OpenEBS.

> La historia de la contenerizacion es la historia de como la industria busco durante 40 anos la forma de aislar procesos sin duplicar hardware. Docker no fue el inventor, sino el estandarizador: tomo conceptos existentes y los envolvio en una experiencia de usuario que任何人都 puede usar.

