# Funcionalidades comunes de los hipervisores Tipo 2

Mas alla de la virtualizacion basica, los hipervisores alojados ofrecen un conjunto de funcionalidades que mejoran la productividad y la experiencia del usuario. Estas funcionalidades convierten a la virtualizacion de herramienta tecnica a herramienta de trabajo diario.

## Carpetas compartidas

Permiten acceder a archivos del sistema anfitrion desde la VM sin necesidad de configurar redes o protocolos de transferencia.

```bash
# VirtualBox: crear carpeta compartida
# Interfaz: Settings > Shared Folders > Add
# CLI:
VBoxManage sharedfolder add "mi-vm" --name "documentos" \
  --hostpath "/home/usuario/documentos" --automount

# Dentro de la VM Linux, montar la carpeta
sudo mkdir -p /mnt/compartida
sudo mount -t vboxsf documentos /mnt/compartida

# Para que se monte automaticamente al iniciar:
# Agregar a /etc/fstab:
# documentos  /mnt/compartida  vboxsf  defaults  0  0
```

## Portapapeles compartido

Copiar y pegar texto, archivos e imagenes entre el host y la VM. Requiere las Guest Additions o VMware Tools instalados.

```
Host (Windows)                    VM (Linux)
┌─────────────┐                  ┌─────────────┐
│ Ctrl+C      │  ← portapapeles → │ Ctrl+V      │
│ Texto copiado│                  │ Texto pegado│
└─────────────┘                  └─────────────┘
```

## Redireccion de dispositivos USB

Conectar dispositivos USB fisicos a la VM, como memorias USB, tarjetas SD, dispositivos IoT o lectores de huellas.

```bash
# VirtualBox: redirigir USB a la VM
# Interfaz: Settings > USB > Add New USB Filter
# CLI:
VBoxManage controlvm "mi-vm" usbattach " Kingston Technology"
```

## Integracion de escritorio

Las herramientas de integracion permiten que la VM se comporta de forma natural en el entorno grafico:

- **Resolucion adaptativa:** alredimensionar la ventana de la VM ajusta la resolucion del SO invitado.
- **Modo seamless:** las ventanas de la VM se mezclan con las del host (solo en algunos hipervisores).
- **Mouse integrado:** el cursor se mueve libremente entre host y VM sin atajos de teclado.
- **Audio compartido:** el audio de la VM se reproduce por los parlantes del host.

## Snapshots y ramificacion de estado

Los snapshots permiten crear multiples puntos de restauracion de una VM. Esto es util para experimentar con diferentes configuraciones sin perder el estado original.

```bash
# Crear snapshots para diferentes escenarios
VBoxManage snapshot "mi-vm" take "estado-base"
VBoxManage snapshot "mi-vm" take "con-base-datos"
VBoxManage snapshot "mi-vm" take "con-servidor-web"

# Restaurar a cualquier punto
VBoxManage snapshot "mi-vm" restore "estado-base"
VBoxManage snapshot "mi-vm" restore "con-base-datos"
```

## Clonacion rapida

Crear VMs identicas a partir de una plantilla base. Utiles para crear entornos de prueba replicables.

```bash
# Clonar VM con linked clone (rapido, comparte disco base)
VBoxManage clonevm "plantilla" --name "copia-01" --register --options link

# Clonar VM con full clone (independiente, copia completa)
VBoxManage clonevm "plantilla" --name "copia-02" --register --options full
```

## Exportar e importar

Las VMs se pueden exportar en formato estandar (OVF/OVA) para transferirlas a otros equipos o hipervisores.

```bash
# Exportar VM en formato OVA
VBoxManage export "mi-vm" --output mi-vm-export.ova

# Importar VM desde archivo OVA
VBoxManage import mi-vm-export.ova
```

> Las funcionalidades de integracion son lo que distingue a un hipervisor Tipo 2 como herramienta de productividad. Carpetas compartidas, portapapeles y resolucion adaptativa eliminan la friccion entre el mundo del host y el de la VM, haciendo que la virtualizacion sea transparente para el usuario final.

