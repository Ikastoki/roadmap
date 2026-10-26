# Instalacion y uso tipico de hipervisores Tipo 2

La instalacion de un hipervisor Tipo 2 es tan sencilla como instalar cualquier aplicacion. No hay configuracion del BIOS, no hay particionado de discos, no hay modo especial de arranque. Descargar, instalar y ejecutar: en minutos se puede tener una maquina virtual funcionando.

## VirtualBox: la opcion mas popular

**Instalacion en Linux:**

```bash
# Agregar repositorios oficiales
sudo apt update
sudo apt install -y virtualbox virtualbox-ext-pack

# Verificar instalacion
VBoxManage --version
# 7.0.14r161095
```

**Instalacion en Windows:**
Descargar el instalador desde [virtualbox.org](https://www.virtualbox.org), ejecutar el `.exe` y seguir el asistente.

**Instalacion en macOS:**
Descargar el `.dmg` desde la pagina oficial, abrir y arrastrar a Applications.

## Crear la primera VM

**Desde la interfaz grafica:**

1. Abrir VirtualBox y hacer clic en "Nueva".
2. Seleccionar nombre, tipo de SO y version.
3. Asignar memoria RAM (recomendado: 2-4 GB para Linux, 4 GB para Windows).
4. Crear disco virtual (VDI, dinamico, 20-50 GB).
5. Seleccionar la ISO de instalacion en Settings > Storage.
6. Iniciar la VM.

**Desde linea de comandos:**

```bash
# Crear VM para Ubuntu 22.04
VBoxManage createvm --name "ubuntu-dev" --ostype Ubuntu_64 --register

# Configurar recursos
VBoxManage modifyvm "ubuntu-dev" --memory 4096 --cpus 2 --vram 128

# Crear disco virtual de 50GB (dinamico)
VBoxManage createmedium disk --filename ~/ubuntu-dev.vdi --size 51200

# Agregar controlador SATA y adjuntar disco
VBoxManage storagectl "ubuntu-dev" --add sata --controller IntelAhci
VBoxManage storageattach "ubuntu-dev" --storagectl sata --port 0 --device 0 \
  --type hdd --medium ~/ubuntu-dev.vdi

# Agregar controlador IDE y adjuntar ISO
VBoxManage storagectl "ubuntu-dev" --add ide --controller PIIX4
VBoxManage storageattach "ubuntu-dev" --storagectl ide --port 0 --device 0 \
  --type dvddrive --medium ~/Descargas/ubuntu-22.04-desktop-amd64.iso

# Configurar red (NAT para acceso a Internet)
VBoxManage modifyvm "ubuntu-dev" --nic1 nat

# Iniciar la VM
VBoxManage startvm "ubuntu-dev"
```

## Instalar Guest Additions

Despues de instalar el SO invitado, instalar las Guest Additions para habilitar carpetas compartidas, portapapeles y resolucion adaptativa.

```bash
# Dentro de la VM Linux:
sudo apt update
sudo apt install -y build-essential dkms linux-headers-$(uname -r)

# Montar el CD de Guest Additions
sudo mount /dev/cdrom /mnt
cd /mnt
sudo ./VBoxLinuxAdditions.run
sudo reboot
```

## Uso tipico diario

```bash
# Iniciar VM en segundo plano
VBoxManage startvm "ubuntu-dev" --type headless

# Apagar VM (shutdown ACPI)
VBoxManage controlvm "ubuntu-dev" acpipowerbutton

# Pausar VM
VBoxManage controlvm "ubuntu-dev" pause

# Tomar snapshot antes de un cambio arriesgado
VBoxManage snapshot "ubuntu-dev" take "antes-de-instalar-npm"

# Restaurar snapshot si algo sale mal
VBoxManage snapshot "ubuntu-dev" restore "antes-de-instalar-npm"
```

## VMware Workstation Player (alternativa gratuita)

```bash
# Linux: instalar VMware Player
sudo apt install ./VMware-Player-*.bundle

# Crear VM desde interfaz o linea de comandos
vmrun clone /path/to/plantilla.vmx /path/to/nueva-vm.vmx
vmrun start /path/to/nueva-vm.vmx
```

> La instalacion de un hipervisor Tipo 2 es la experiencia mas directa de virtualizacion. Sin configuraciones complejas ni hardware especializado, un desarrollador puede tener un laboratorio completo de multiples sistemas operativos ejecutandose en su portatil en cuestión de minutos.

