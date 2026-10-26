# Caracteristicas de los hipervisores nativos

Los hipervisores de Tipo 1 estan disenados para entornos de produccion donde la disponibilidad, el rendimiento y la seguridad son criticos. Sus caracteristicas los convierten en la piedra angular de los centros de datos modernos, permitiendo ejecutar cientos de VMs sobre un cluster de servidores fisicos con gestion centralizada.

## Gestion centralizada

En entornos empresariales, los hipervisores se despliegan en multiples servidores fisicos (hosts) y se gestionan desde una plataforma centralizada.

- **VMware vCenter:** plataforma de gestion para clusters ESXi. Permite crear, mover, monitorear y eliminar VMs desde una unica interfaz.
- **Proxmox VE:** interfaz web para gestionar un cluster de nodos KVM.
- **XenServer (XCP-ng):** gestion centralizada de hosts Xen.
- **System Center VMM (Microsoft):** gestion de hosts Hyper-V.

```
vCenter Server (centralizado)
├── Host ESXi 1 (8 VMs)
├── Host ESXi 2 (12 VMs)
├── Host ESXi 3 (10 VMs)
└── Host ESXi 4 (6 VMs)
Total: 36 VMs gestionadas desde un solo punto
```

## Migracion en vivo (Live Migration)

Una de las funcionalidades mas poderosas: mover una VM de un servidor fisico a otro sin apagarla ni interrumpir el servicio. Los usuarios no notan nada.

**Como funciona:**
1. El hipervisor origen copia la memoria de la VM al host destino.
2. Sincroniza los cambios de memoria que ocurren durante la transferencia.
3. Transfiere el control al host destino.
4. La VM continua ejecutandose en el nuevo servidor.

**Requisitos:** almacenamiento compartido (SAN, NAS, vSAN) o replicacion de disco.

```bash
# Migracion en vivo con vCenter (ejemplo conceptual)
# vCenter detecta carga desbalanceada y migra VMs automaticamente
# O manualmente desde la interfaz:
# Click derecho sobre VM > Migrar > Seleccionar host destino
```

## Alta disponibilidad (HA)

Si un servidor fisico falla, el hipervisor reinicia automaticamente las VMs que alojaba en otros servidores del cluster. No es una migracion en vivo (hay un breve downtime), pero garantiza la recuperacion rapida.

```
Cluster de alta disponibilidad
├── Host 1: VM-A, VM-B, VM-C
├── Host 2: VM-D, VM-E, VM-F
└── Host 3: (reserva para failover)

Si Host 1 falla:
VM-A, VM-B, VM-C se reinician en Host 2 o Host 3
```

## Balanceo de carga

Distribuye las VMs entre los servidores fisicos para equilibrar el uso de CPU, memoria y red. Esto evita que un servidor este sobrecargado mientras otro esta ocioso.

- **DRS (Distributed Resource Scheduler) en VMware:** analiza la carga y sugiere o ejecuta migraciones automaticas.
- **Politicas de afinidad:** reglas que controlan donde se ejecutan las VMs (ej. "esta VM siempre en Host 1").

## Snapshots

Capturan el estado completo de una VM en un momento dado: memoria, disco, configuracion. Permiten revertir la VM a ese estado posteriormente.

```bash
# Crear snapshot en VMware ESXi (PowerCLI)
New-Snapshot -VM "servidor-web" -Name "antes-de-actualizacion" -Description "Snapshot preventivo"

# Revertir a un snapshot
Set-VM -VM "servidor-web" -Snapshot "antes-de-actualizacion" -Confirm:$false
```

> Los hipervisores nativos no solo virtualizan hardware: transforman centros de datos completos en entornos弹性 y resilientes. La combinacion de migracion en vivo, alta disponibilidad y balanceo de carga crea una infraestructura donde el hardware falla pero el servicio no se interrumpe.

