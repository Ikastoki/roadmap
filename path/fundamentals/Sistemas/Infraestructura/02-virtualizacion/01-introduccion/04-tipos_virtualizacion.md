# Tipos de virtualizacion

La virtualizacion no se limita a crear maquinas virtuales. Existen multiples variantes que abstraen diferentes capas de la infraestructura: hardware, red, almacenamiento, escritorio y aplicaciones. Cada tipo resuelve un problema especifico y se implementa con tecnologias distintas.

## Virtualizacion de hardware

Es la forma mas conocida y extendida. Crea maquinas virtuales completas, cada una con su propio sistema operativo, ejecutadas sobre un hipervisor.

- **Uso principal:** consolidacion de servidores, entornos de prueba, ejecucion de software legacy.
- **Tecnologias:** VMware ESXi, KVM, Hyper-V, VirtualBox, Xen.
- **Caracteristica clave:** cada VM tiene su propio kernel, sistema de archivos y configuracion.

```
Servidor fisico
├── Hipervisor (ESXi/KVM)
│   ├── VM 1: Windows Server 2022 (IIS)
│   ├── VM 2: Ubuntu 22.04 (Apache)
│   ├── VM 3: CentOS 7 (legacy app)
│   └── VM 4: FreeBSD (firewall)
└── Hardware compartido
```

## Virtualizacion de red

Abstrae la red fisica para crear redes logicas independientes sobre la misma infraestructura fisica.

- **VLANs:** segmentacion logica de redes en el nivel 2 del modelo OSI.
- **SDN (Software-Defined Networking):** control centralizado del comportamiento de la red mediante software.
- **Redes virtuales privadas (VPC):** redes aisladas dentro de la nube.
- **Virtual switches (vSwitch):** switches virtuales que conectan VMs entre si y con la red fisica.

```bash
# Crear una VLAN en Linux
sudo ip link add link eth0 name eth0.100 type vlan id 100
sudo ip addr add 192.168.100.1/24 dev eth0.100
sudo ip link set eth0.100 up
```

## Virtualizacion de almacenamiento

Abstrae los dispositivos de almacenamiento fisicos para presentarlos como recursos logicos.

- **SAN virtualizada:** red de almacenamiento que presenta discos logicos a los servidores.
- **NAS virtualizada:** archivos compartidos sobre una red, gestionados por software.
- **SDS (Software-Defined Storage):** gestion de almacenamiento mediante software, independiente del hardware.
- **Discos virtuales:** archivos que emulan discos fisicos para las VMs.

```
Almacenamiento fisico (discos DAS)
├── Capa de abstraccion (Ceph, VMware vSAN)
│   ├── Datastore 1 (para VMs de produccion)
│   ├── Datastore 2 (para VMs de desarrollo)
│   └── Datastore 3 (para backups)
```

## Virtualizacion de escritorio (VDI)

Centraliza los escritorios de los usuarios en el servidor. Los usuarios acceden a su escritorio virtual desde cualquier dispositivo a traves de la red.

- **Uso principal:** empresas con trabajadores moviles, call centers, entornos de seguridad.
- **Tecnologias:** VMware Horizon, Citrix Virtual Apps, Apache Guacamole.
- **Ventaja:** los datos nunca salen del centro de datos, solo se transmite la imagen del escritorio.

## Virtualizacion de aplicaciones

Aísla aplicaciones individuales sin necesidad de un sistema operativo completo. Es la evolucion natural hacia los contenedores.

- **App-V (Microsoft):** empaqueta aplicaciones Windows para ejecutarlas aisladas.
- **Flatpak/Snap (Linux):** empaquetado de aplicaciones con dependencias incluidas.
- **Docker:**虽然 technically es un contenedor, es la evolucion de la virtualizacion de aplicaciones.

## Resumen de tipos

| Tipo | Que abstrae | Ejemplo |
|---|---|---|
| Hardware | CPU, memoria, disco completo | VMware ESXi, KVM |
| Red | switches, routers, firewalls | VLAN, SDN, VPC |
| Almacenamiento | discos, archivos, bloques | Ceph, vSAN, NAS |
| Escritorio | escritorio completo del usuario | VMware Horizon, Citrix |
| Aplicacion | una o varias aplicaciones | Docker, Snap, App-V |

> Cada tipo de virtualizacion ataca un problema distinto de eficiencia o aislamiento. El hardware consolida servidores; la red segmenta trafico; el almacenamiento centraliza datos; el escritorio moviliza usuarios; la aplicacion aislada elimina conflictos de dependencias. Juntos, cubren toda la pila de infraestructura.

