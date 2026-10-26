# Caracteristicas de los hipervisores alojados

Los hipervisores de Tipo 2 estan disenados para ser faciles de instalar, configurar y utilizar. Su principal ventaja es la accesibilidad: no requieren hardware dedicado, no eliminan el sistema operativo existente y proporcionan una experiencia de virtualizacion integrada con el entorno de trabajo habitual.

## Facilidad de instalacion

Se instalan como cualquier otra aplicacion del sistema operativo anfitrion. No requieren particionar discos, cambiar el orden de arranque ni reiniciar en un modo especial.

```bash
# Linux: instalar VirtualBox desde repositorios oficiales
sudo apt update
sudo apt install virtualbox virtualbox-ext-pack

# Linux: instalar VMware Workstation Pro
sudo apt install ./VMware-Workstation-Full-*.bundle

# macOS: instalar Parallels desde el DMG
# Doble clic en el instalador, arrastrar a Applications
```

## Herramientas de integracion (Guest Additions / VMware Tools)

Son controladores y agentes que se instalan dentro de la VM invitada para mejorar significativamente la experiencia:

- **Controladores de video:** resolucion adaptativa, aceleracion 3D basica.
- **Sincronizacion de tiempo:** la VM mantiene la hora del host.
- **Copia/pega:** portapapeles compartido entre host y VM.
- **Carpetas compartidas:** acceder a archivos del host desde la VM.
- **Arrastrar y soltar:** mover archivos entre host y VM.
- **Control de mouse integrado:** el cursor sale de la ventana sin atajos de teclado.

```bash
# VirtualBox: instalar Guest Additions desde la VM
# Montar el CD virtual desde la interfaz, luego dentro de la VM:
sudo mount /dev/cdrom /mnt
cd /mnt
sudo ./VBoxLinuxAdditions.run

# VMware: instalar VMware Tools desde la interfaz
# VM > Install VMware Tools
# Dentro de la VM:
sudo mount /dev/cdrom /mnt
cd /mnt
sudo tar xzf VMwareTools-*.tar.gz -C /tmp
cd /tmp/vmware-tools-distrib
sudo ./vmware-install.pl
```

## Soporte para snapshots

Los hipervisores Tipo 2 permiten crear snapshots (capturas del estado de la VM) que pueden restaurarse en cualquier momento.

```bash
# VirtualBox: crear snapshot
VBoxManage snapshot "mi-vm" take "antes-de-actualizacion"

# VirtualBox: listar snapshots
VBoxManage snapshot "mi-vm" list

# VirtualBox: restaurar snapshot
VBoxManage snapshot "mi-vm" restore "antes-de-actualizacion"
```

Los snapshots son especialmente utiles antes de:
- Instalar software que podria romper el sistema.
- Modificar configuraciones criticas.
- Probar cambios experimentales.

## Clonacion de VMs

Crear copias completas de una VM en segundos:

```bash
# VirtualBox: clonar VM
VBoxManage clonevm "plantilla-ubuntu" --name "servidor-web-01" --register
VBoxManage clonevm "plantilla-ubuntu" --name "servidor-web-02" --register
```

## Multiples formatos de disco

- **VDI (VirtualBox Disk Image):** formato nativo de VirtualBox.
- **VMDK (Virtual Machine Disk):** formato VMware, ampliamente compatible.
- **VHD/VHDX (Virtual Hard Disk):** formato Microsoft Hyper-V.
- **QCOW2:** formato de QEMU, soporta snapshots y thin provisioning.

```bash
# Convertir entre formatos
VBoxManage clonformat --format VMDK mi-vm.vdi mi-vm.vmdk
```

> Las caracteristicas de los hipervisores alojados los convierten en la herramienta ideal para aprendizaje, desarrollo y pruebas. La combinacion de facilidad de uso, snapshots, clonacion y herramientas de integracion permite experimentar con multiples sistemas operativos sin configuraciones complejas ni hardware dedicado.

