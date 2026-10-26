# Instalacion y operacion de hipervisores Tipo 1

Instalar un hipervisor bare-metal es fundamentalmente diferente a instalar un sistema operativo convencional. No hay escritorio, no hay navegador, no hay aplicaciones: el hipervisor es un sistema operativo minimalista disenado exclusivamente para gestionar VMs. La instalacion implica preparar el servidor, configurar almacenamiento y red, y acceder a la interfaz de gestion.

## Preparacion del servidor

**Requisitos minimos:**
- CPU con extensiones de virtualizacion (Intel VT-x o AMD-V).
- Al menos 4 GB de RAM (8 GB recomendados para empezar).
- Disco SSD o NVMe para el sistema del hipervisor.
- Al menos una tarjeta de red.
- ISO del hipervisor en USB o acceso a red para PXE boot.

```bash
# Verificar compatibilidad del CPU
grep -E "vmx|svm" /proc/cpuinfo
# vmx: Intel VT-x habilitado
# svm: AMD-V habilitado
```

## Instalacion de VMware ESXi

1. Descargar la ISO de ESXi desde VMware Customer Connect.
2. Crear un USB booteable con Rufus o `dd`.
3. Arrancar el servidor desde el USB.
4. Seguir el asistente de instalacion (seleccionar disco, configurar root password).
5. Reiniciar. El servidor arrancara en ESXi.

```
Pantalla de ESXi despues de la instalacion:
┌─────────────────────────────────────────┐
│  VMware ESXi 8.0                        │
│  IP: 192.168.1.100                      │
│  <F2> Customize System                  │
│  <F12> Shut Down/Restart                │
└─────────────────────────────────────────┘
```

**Acceso a la interfaz web:** abrir `https://192.168.1.100` en un navegador desde otro equipo. Se descarga la clienta vSphere para gestionar VMs.

## Instalacion de Proxmox VE

Proxmox combina KVM (virtualizacion) y LXC (contenedores) con una interfaz web completa.

1. Descargar la ISO de Proxmox VE.
2. Crear USB booteable.
3. Arrancar y seguir el asistente.
4. Acceder a `https://ip-servidor:8006`.

```bash
# Proxmox se instala sobre Debian, por lo que puedes usar apt
# Despues de la instalacion, actualizar el sistema
apt update && apt upgrade -y
```

## Configuracion inicial comun

**Almacenamiento (datastores):**

```bash
# En ESXi, crear datastore desde la interfaz web o CLI
# CLI de ESXi:
esxcli storage filesystem mount /dev/sdb1

# En Proxmox, el almacenamiento local ya esta configurado
# Agregar almacenamiento NFS compartido:
pvesm add nfs nfs-store --server 192.168.1.200 --export /srv/nfs
```

**Red:**

```bash
# Configurar red en ESXi (CLI)
esxcli network vswitch standard add --vswitch-name=vSwitch0
esxcli network vswitch standard uplink add --uplink-name=vmnic0 --vswitch-name=vSwitch0
esxcli network vswitch standard portgroup add --portgroup-name=VM-Network --vswitch-name=vSwitch0
```

## Crear la primera VM

```bash
# En ESXi (PowerCLI)
New-VM -Name "ubuntu-server" -VMHost 192.168.1.100 `
  -NumCpu 2 -MemoryGB 4 -DiskGB 50 `
  -Datastore "datastore1" -NetworkName "VM-Network"

# En Proxmox
qm create 100 --name ubuntu-server --memory 4096 --cores 2 \
  --scsihw virtio-scsi-pci --scsi local-lvm:50 \
  --net0 virtio,bridge=vmbr0
```

## Monitoreo basico

```bash
# En ESXi, ver uso de recursos desde CLI
esxcli system stats get

# En Proxmox
qm status 100          # Estado de una VM
pvesh get /nodes/$(hostname)/status  # Uso de recursos del nodo
```

> La instalacion de un hipervisor Tipo 1 es el punto de entrada a la virtualizacion empresarial. Una vez que el hipervisor esta operativo, el servidor fisico se transforma en una plataforma que puede ejecutar docenas de sistemas operativos, cada uno con su propio proposito, todos gestionados desde un solo punto.

