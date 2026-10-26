# Migracion y portabilidad

Una de las grandes ventajas de la virtualizacion es que las maquinas virtuales son portables. Una VM es esencialmente un conjunto de archivos que se pueden copiar, mover, exportar e importar entre servidores, hipervisores e incluso entre centros de datos. Esta portabilidad permite migrar VMs sin reinstalar el SO ni reconfigurar las aplicaciones.

## Migracion en vivo (Live Migration)

Mover una VM de un servidor fisico a otro sin apagarla ni interrumpir el servicio. Los usuarios no notan nada.

**Requisitos:**
- Almacenamiento compartido (SAN, NAS, vSAN) o replicacion de disco.
- Red de alta velocidad entre los servidores origen y destino.
- Mismo tipo de hipervisor en ambos servidores.

**Proceso conceptual:**
1. El host origen copia la memoria de la VM al host destino.
2. Sincroniza los cambios de memoria que ocurren durante la transferencia.
3. Transfiere el control al host destino.
4. La VM continua en el nuevo servidor sin interrupcion.

```bash
# VMware vCenter: migracion en vivo (PowerCLI)
Move-VM -VM "servidor-web" -Destination "esxi-host-02"

# Proxmox: migracion en vivo
qm migrate 100 node2 --online
```

## Migracion apagada (Cold Migration)

Apagar la VM, mover los archivos y encenderla en el nuevo servidor. Es mas simple pero implica downtime.

```bash
# VirtualBox: exportar VM
VBoxManage export "mi-vm" --output mi-vm.ova

# En el nuevo equipo: importar
VBoxManage import mi-vm.ova
```

## Exportar e importar en formatos estandar

**OVF (Open Virtualization Format):** estandar abierto para empaquetado de VMs. Incluye descriptors XML y archivos de disco.

**OVA (Open Virtualization Archive):** archivo OVF empaquetado como tar.gz. Un unico archivo con todo lo necesario.

```bash
# VirtualBox: exportar a OVA
VBoxManage export "mi-vm" --output mi-vm-completo.ova

# VirtualBox: importar desde OVA
VBoxManage import mi-vm-completo.ova

# VMware: exportar a OVF
# vCenter: File > Export > OVF

# QEMU/KVM: convertir disco a formato estandar
qemu-img convert -f qcow2 -O vmdk mi-disco.qcow2 mi-disco.vmdk
```

## Conversion de formatos

```bash
# Convertir VDI (VirtualBox) a VMDK (VMware)
VBoxManage clonformat --format VMDK mi-vm.vdi mi-vm.vmdk

# Convertir VMDK a QCOW2 (KVM)
qemu-img convert -f vmdk -O qcow2 mi-vm.vmdk mi-vm.qcow2

# Convertir VHD (Hyper-V) a VMDK
qemu-img convert -f vpc -O vmdk mi-vm.vhd mi-vm.vmdk
```

## Compatibilidad entre hipervisores

La compatibilidad directa entre hipervisores es limitada. Cada plataforma usa formatos de disco y configuraciones especificas. La conversion de formatos es necesaria para migrar entre plataformas diferentes.

| Origen | Destino | Metodo |
|---|---|---|
| VMware | VirtualBox | Exportar OVA > Importar |
| VirtualBox | VMware | Exportar OVA > Importar |
| VMware | KVM | Convertir VMDK a QCOW2 |
| KVM | VMware | Convertir QCOW2 a VMDK |
| Hyper-V | VMware | Convertir VHD a VMDK |

## Consideraciones de portabilidad

- **Drivers:** el SO invitado puede necesitar drivers diferentes en el nuevo hipervisor.
- **IP y red:** la configuracion de red puede cambiar al migrar.
- **Licencias:** algunas licencias de SO estan atadas a hardware especifico.
- **Herramientas de integracion:** VMware Tools no funcionan en VirtualBox y viceversa; hay que reinstalar las herramientas correctas.

> La migracion y portabilidad son ventajas competitivas de la virtualizacion sobre el hardware fisico. Migrar un servidor fisico a otro equipo requiere reinstalar todo; migrar una VM es copiar un conjunto de archivos. Esta diferencia es lo que permite la alta disponibilidad, la recuperacion ante desastres y la flexibilidad en la gestion de centros de datos.

